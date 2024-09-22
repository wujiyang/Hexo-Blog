---
title: BigCache解析
date: 2024-08-10 17:50:28
index_img: /img/BigCache解析/0.png
tags:
    - BigCache
    - 本地缓存
categories:
    - 系统设计
---

BigCache是一个快速，支持并发访问，自淘汰的内存型缓存，可以在存储大量元素的基础上依然保持高性能。BigCache将元素保存在堆上的同时避免了GC的开销。
<!-- more -->  

## 1. 简介
缓存是系统提升并发能力、降低时延的利器，根据存储介质和使用场景，我们分为本地缓存与分布式缓存两种：
- 本地缓存：一般在进程内，最简单的sync.Map就可以是一个并发安全的本地缓存。常见的有LocalCache、BigCache等。
- 分布式缓存：一般会用到Redis/MemCached等分布式内存数据库实现，想比如本地缓存，分布式数据库增加了网络开销  

BigCache是一个快速，支持并发访问，自淘汰的内存型缓存，可以在存储大量元素的基础上依然保持高性能。BigCache将元素保存在堆上的同时避免了GC的开销。

源码地址：https://github.com/allegro/bigcache   
本文示例代码取自版本 v1.2.1  

## 2. 设计思想
### 2.1 整体设计
- 多： 缓存的元素数量非常大，可以达到百万级或千万级。
- 快： 对延迟有非常高的要求，平均延迟要求在5毫秒以内。支持10k rps级别的访问速度。
- 稳： 99.9分位延迟应在10毫秒左右，99.999分位延迟应在400毫秒左右。

目前有许多开源的cache库，大部分都是基于map实现的，例如go-cache,ttl-cache等。bigcache明确指出，当数据量巨大时，直接基于map实现的cache库将出现严重的性能问题，这也是他们设计了一个全新的cache库的原因。

- 核心设计思想
    - 数据分片存储，以降低锁冲突并提升并发量。
    - 避免在map中存储指针，从而避免在GC时对map进行遍历扫描。
    - 采用FIFO式的Ring Buffer设计，简化整体内存设计逻辑。


![](/img/BigCache解析/1.png)

### 2.2 数据分片 Shard
使用map做缓存时，实现并发安全的方式就是加一把锁。但是如果数据过多时，并发请求会导致较多的锁冲突，为了解决这种问题，BigCache采用了分片的方式降低锁粒度。在声明一个BigCache时，表面上看是一个结构体，实际上是一个数组，底层分成了N个互不关联的部分。
```go
type BigCache struct {
	shards       []*cacheShard
	lifeWindow   uint64
	clock        clock
	hash         Hasher
	config       Config
	shardMask    uint64
	maxShardSize uint32
	close        chan struct{}
}
```

在Set或者Get数据时，先对key计算hash值，根据hash值取余得到目标shard，之后所有的读写操作都是在各自的shard上进行。
```go
func (c *BigCache) Set(key string, entry []byte) error {
	hashedKey := c.hash.Sum64(key)
	shard := c.getShard(hashedKey)
	return shard.set(key, hashedKey, entry)
}
```
在计算分片数时，取余操作使用的是位运算，所有分片数要设置为2的幂次方。
```go
func (c *BigCache) getShard(hashedKey uint64) (shard *cacheShard) {
	return c.shards[hashedKey&c.shardMask]
}
// shardMask = shardNum - 1
```
- 特点：
    - 减少锁冲突，提升并发量：当一个shard被加上Lock的时候，其他shard的读写不受影响。
    - shard一旦建好，将不再改变，无需考虑shard变化时的数据迁移问题，shard之间操作无需加锁。
    - shard个数必须是2的平方数。对2的平方数取余可以改成位运算，会比传统的%快很多


### 2.3 Map GC优化
在golang中，map的key和value一旦涉及到指针类型，在GC的时候就会触发遍历扫描，当数据量级很大时， GC延迟严重。
在bigcache的设计中，map定义为map[uint64]uint32 ，避免了存储任何指针。结构定义如下:
```go
type cacheShard struct {
	hashmap     map[uint64]uint32
	entries     queue.BytesQueue
	...
}
```
其中：
- hashmap的key是cache key的hash值，而value仅仅是个uint32。value存储的不再是实际值，而是value在 BytesQueue中的索引。
- entries存储的是实际value值，它是一个ring buffer的内存结构，本质上就是个超大的[]byte数组，里面存放了所有的原始数据。每个原始数据就存放在这个大[]byte数组中的其中一段。
之所以用一个大的[]byte数组和ring buffer结构，除了方便管理和复用内存之外，一个更重要的原因是：对于[]byte数组, GC时只用看做一个变量扫描，无需再遍历全部数组。这样又避免了海量数据对GC造成的负担。

### 2.4 BytesQueue内存
BigCache中的每一个shard都采用类似RingBuffer的结构
- 对于map中每个元素而言，key存储的是 cache key的hash值，value存储的是实际值在byte数组中的位置。
- 数据存储完全采用FIFO的形式，所有数据的新增、包括老数据的修改，都是直接追加到数组的后面，然后修改map的位置值即可。
![](/img/BigCache解析/2.png)

具体到每一个数据的存储，并不是完全只存储val值，会把一些相关的信息一起存储，构成数据header和body，，一个header和一个body构成一个完整的条目，其中：
- Block Size：标记header和Body，也就是Entry的总长度
- header：包括 插入时间戳，key的hash值，key的长度， header的长度固定
- body：key的原始值，value的原始值
- 有了BlockSize、Header Length以及Key的长度，就可以确定value的边界并正常进行数据读取了。
![](/img/BigCache解析/3.png)

```go
const (
	// header中的时间戳、hash值和key长度占用字节数固定，可以方便找到key原始值的起始位置
	timestampSizeInBytes = 8       // Number of bytes used for timestamp
	hashSizeInBytes      = 8       // Number of bytes used for hash
	keySizeInBytes       = 2       // Number of bytes used for size of entry key
	headersSizeInBytes   = timestampSizeInBytes + hashSizeInBytes + keySizeInBytes // Number of bytes used for all headers
)

func wrapEntry(timestamp uint64, hash uint64, key string, entry []byte, buffer *[]byte) []byte {
	keyLength := len(key)
	blobLength := len(entry) + headersSizeInBytes + keyLength

	if blobLength > len(*buffer) {
		*buffer = make([]byte, blobLength)
	}
	blob := *buffer
	
	// 按照小端序的方式，顺序写入时间戳、key的hash值、key的长度、key的实际值以及value的实际值
	binary.LittleEndian.PutUint64(blob, timestamp)
	binary.LittleEndian.PutUint64(blob[timestampSizeInBytes:], hash)
	binary.LittleEndian.PutUint16(blob[timestampSizeInBytes+hashSizeInBytes:], uint16(keyLength))
	copy(blob[headersSizeInBytes:], key)
	copy(blob[headersSizeInBytes+keyLength:], entry)

	return blob[:blobLength]
}
```
- RingBuffer满时，bigCache如何处理？
    - 如果entries queue.BytesQueue 未达到设定的HardMaxCacheSize（最大内存上限），或者无HardMaxCacheSize要求，则直接扩容queue.BytesQueue 直到达到上限。不过扩容的时候，是创建了一个新的空[]byte数组，把原有数据copy过去。
    - 如果内存已达上限，无法继续扩容，则会尝试删除最旧数据（无论是否过期），直至可以将数据放到BytesQueue中。如果这个时候新数据非常大，可能会为此淘汰掉许多旧数据。


### 2.5 锁冲突
BigCache使用Key的hash值查找数据，不可避免的会导致hash冲突问题。锁冲突在写入和读取的过程中都有可能发生。下面分别分析Set和Get的过程，顺便看看锁冲突是如何解决的。
- Set流程
    - 计算key的hash值，得到对应的shard
    - 将key和value等信息序列化成指定格式的[]byte, push到BytesQueue中。
    - 根据BytesQueue返回的偏移量(也就是数组下标)，将key(hash值)和value(数组下标)设置hashmap中。
```go
func (s *cacheShard) set(key string, hashedKey uint64, entry []byte) error {
	currentTimestamp := uint64(s.clock.epoch())

	s.lock.Lock()
	// 从这里可以看到，如果发现存在已有的hashKey（可能是冲突，也可能是本来就存在），bigcache并不会做特殊处理，而是直接将原本的位置reset，并将新的值push到最后，然后替换这个key的下标
	if previousIndex := s.hashmap[hashedKey]; previousIndex != 0 {
		if previousEntry, err := s.entries.Get(int(previousIndex)); err == nil {
			// resetKeyFromEntry 会将条目中key的hash值置零
			resetKeyFromEntry(previousEntry)
		}
	}

	if oldestEntry, err := s.entries.Peek(); err == nil {
		s.onEvict(oldestEntry, currentTimestamp, s.removeOldestEntry)
	}
	
	// 构建一个数据条目
	w := wrapEntry(currentTimestamp, hashedKey, key, entry, &s.entryBuffer)

	for {
		// 将数据条目push到最后，并修改hashmap的值，就完成了增加或者更新
		if index, err := s.entries.Push(w); err == nil {
			s.hashmap[hashedKey] = uint32(index)
			s.lock.Unlock()
			return nil
		}
		if s.removeOldestEntry(NoSpace) != nil {
			s.lock.Unlock()
			return fmt.Errorf("entry is bigger than max shard size")
		}
	}
}
```
- Get流程
    - 读数据，hash冲突时直接返回数据不存在，可能这也是存储key实际值的意义所在。
```go
func (s *cacheShard) get(key string, hashedKey uint64) ([]byte, error) {
	s.lock.RLock()
	itemIndex := s.hashmap[hashedKey]

	// bigcache 中， ringbuffer 的第 0 位并不用来存放任何数据，
	// 如果发现 分片 map 中得到数据的 index 为 0，就可以直接认为没有对应的缓存数据
	if itemIndex == 0 {
		s.lock.RUnlock()
		s.miss()
		return nil, ErrEntryNotFound
	}
	
	// 从ringbuff中获取到指定key的条目数据
	wrappedEntry, err := s.entries.Get(int(itemIndex))
	if err != nil {
		s.lock.RUnlock()
		s.miss()
		return nil, err
	}
	// 从数据中解析出实际存储的key
	// 如果实际存储的key跟查询的key不一致，说明产生了hash冲突，直接返回空
	if entryKey := readKeyFromEntry(wrappedEntry); key != entryKey {
		if s.isVerbose {
			s.logger.Printf("Collision detected. Both %q and %q have the same hash %x", key, entryKey, hashedKey)
		}
		s.lock.RUnlock()
		s.collision()
		return nil, ErrEntryNotFound
	}
	entry := readEntry(wrappedEntry)
	s.lock.RUnlock()
	s.hit()
	return entry, nil
}
```


### 2.6 数据删除
BigCache的删除比较简单，本质上只是将key从map中删除，并没有实际删除BytesQueue中的数据，核心逻辑只有两行。实际值的删除会等到数据清理或者Buffer满的时候淘汰掉。
```go
...
// 从hashmap中删除key，使数据不可读
delete(s.hashmap, hashedKey)
...
// 将entry条目的key的hash值置零
resetHashFromEntry(wrappedEntry)
...
```
### 2.7 过期淘汰
BigCache中的数据在删除数据或者Set数据时，如果碰到就值，只是删除或者修改hashmap中的值，并不会实际删除数据。实际的数据删除发生在过期淘汰时。

📌 <font color="red">BigCache没有开放单个元素的可过期时间，所有元素的cache时长都是一样的，这就意味着所有元素的过期时间在队列中天然有序。</font>    
```go
func (s *cacheShard) cleanUp(currentTimestamp uint64) {
	s.lock.Lock()
	for {
		// 只要存在元素，就逐个便利
		if oldestEntry, err := s.entries.Peek(); err != nil {
			break
		// 直至某个元素不过期为止
		} else if evicted := s.onEvict(oldestEntry, currentTimestamp, s.removeOldestEntry); !evicted {
			break
		}
	}
	s.lock.Unlock()
}
```

## 3. 注意事项
### 3.1 数据过期处理
在bigcache中，LifeWindow和CleanWindow是两个用于缓存管理的重要参数，但它们有不同的作用。
- LifeWindow 是缓存条目的生存时间，决定了条目在多长时间内会被视为过期。
- CleanWindow 是清理过程的时间间隔，决定了过期条目实际被清除的频率。

这种机制使得bigcache在设计上允许读取过期数据。它不会主动阻止访问已经过期的缓存条目，除非这些条目在CleanWindow期间被清理掉。

bigcache的这种设计是为了优化性能，因为在每次访问缓存时都检查条目是否过期会带来额外的性能开销。通过异步清理过期条目，bigcache能够在不影响访问速度的情况下提供缓存功能。

如果对数据的时效性要求很高，需要手动在读取后检查数据是否过期，代码示例如下：
```go
item, err := cache.Get("myKey")
if err != nil {
    // Handle the error
}

timestamp := extractTimestamp(item) // 解析数据中的时间戳
if time.Since(timestamp) > cacheLifeWindow {
    // Data is considered stale
    // Handle the stale data case
}
```

## 4. 参考文献
https://juejin.cn/post/7107635176263385118

https://www.cyhone.com/articles/bigcache/ 

https://mp.weixin.qq.com/s/e-ku-fLsLk0Ok2eTkdx6Dw

https://juejin.cn/post/7258840980428750885

https://xiaorui.cc/archives/7385