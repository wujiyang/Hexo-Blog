---
title: Redis分布式锁
date: 2023-12-23 18:13:13
index_img: /img/Redis分布式锁/1.png
tags:
    - Redis
    - 分布式锁
categories:
    - Redis
---

分布式锁，就是控制分布式系统不同进程共同访问共享资源的一种锁的实现。如果不同的系统或同一个系统的不同主机之间共享了某个临界资源，往往需要互斥来防止彼此干扰，以保证一致性。 

<!-- more -->   

## Redis分布式锁的实现方式
>在线redis环境：https://try.redis.io/   
### 1. 什么是分布式锁
分布式锁就是，控制分布式系统不同进程共同访问共享资源的一种锁的实现。如果不同的系统或同一个系统的不同主机之间共享了某个临界资源，往往需要互斥来防止彼此干扰，以保证一致性。   
分布式锁一般具有以下特征:  
- 互斥性：任意时刻只有一个客户端可持有
- 超时释放：持有锁超时可以及时释放，防止死锁和不必要的资源浪费  
- 可重入性：一个线程获取锁之后，还可以再次对其请求加锁
- 高性能高可用：加锁和解锁需要开销尽量低，同时也要保证高可用 
- 安全性：锁只能被持有的客户端删除，不能被其他客户端删除    

<!-- {% asset_img 1.png %} -->
![](/img/Redis分布式锁/1.png)

### 2. SETNX  
一提到使用Redis作为分布式锁，大家耳熟能详的就会想到```SETNX + EXPIRE```。即先用```setnx```来抢锁，然后再用```expire```设置一个过期时间。
> SETNX是 SET IF NOT EXISTS的简称，顾名思义就是不存在就设置  
> 命令使用：SETNX key value  
> 如果key不存在，设置成功返回1，如果key已经存在，设置失败返回0

但是直接使用SETNX + EXPIRE 指令，会存在一些问题：
- 原子性：setnx 和 exipre是两个命令，如果设置过期时间出错，就会导致长时间持有锁得不到释放。
- 误删除问题：假设线程a执行完后，主动去释放锁。但它不知道当前的锁可能是线程b持有的（线程a去释放锁时，有可能过期时间已经到了，此时线程b进来占有了锁）。那线程a就把线程b的锁释放掉了，但是线程b临界区业务代码可能都还没执行完成。  

#### 2.1 原子性
为解决原子性问题，常见的有两种方式:  

##### 2.1.1 使用Lua脚本 
lua脚本在执行过程中，是可以保证原子处理的，可以将setnx和expire两个命令都放到lua脚本中进行操作。  
``` lua 
if redis.call('setnx',KEYS[1],ARGV[1]) == 1 then
   redis.call('expire',KEYS[1],ARGV[2])
else
   return 0
end;
```
##### 2.1.2 使用SET的扩展命令 
虽然```SETNX```和```EXPIRE```两个指令是独立的，但是单独的一个SET命令是却是原子的，可以使用SET的参数扩展功能实现NX和EXPIRE的能力  
```SET key value [EX seconds] [PX millseconds] NX```  
>SET key value [EX seconds] [PX milliseconds] NX  
>NX :表示key不存在的时候，才能set成功，也即保证只有第一个客户端请求才能获得锁，而其他客户端请求只能等其释放锁，才能获取。  
>EX seconds :设定key的过期时间，时间单位是秒。  
>PX milliseconds: 设定key的过期时间，单位为毫秒  


在实际的开发过程中，我们一般使用第三方库，不会直接执行redis的cmd命令。可以在使用第三方的sdk时，查看sdk是否已经提供了一些原子的操作，避免自己写原生lua脚本或者命令。比如go-redis中，SETNX + EXPIRE已经可以在一个客户端操作中完成：  
``` go
// Lock 使用 SETNX实现加锁
func (c *Client) Lock(key, value string) error {
	ret := c.client.SetNX(key, value, time.Minute)
	if ret.Val() {
		fmt.Println("加锁成功")
		return nil
	}
	return errors.New("加锁失败")
}

// Redis `SET key value [expiration] NX` command.
//
// Zero expiration means the key has no expiration time.
func (c *cmdable) SetNX(key string, value interface{}, expiration time.Duration) *BoolCmd {}

```

#### 2.2 误删除
既然锁可能会被别的线程删除，那给锁的value值设置一个标记当前的线程唯一值即可。在删除的时候首先校验值是否相等，只有相等的情况下才可以删除锁。  
同样的，为保证一致性，值的比较和删除操作需要保证原子性。 
``` lua 
if redis.call('get',KEYS[1]) == ARGV[1] then 
   return redis.call('del',KEYS[1]) 
else
   return 0
end;
```
### 3. Redisson  

参考文章：  
- [https://blog.csdn.net/qq_34826261/article/details/126177704](https://blog.csdn.net/qq_34826261/article/details/126177704)  
- [https://zhuanlan.zhihu.com/p/135864820](https://zhuanlan.zhihu.com/p/135864820)  
- [https://juejin.cn/post/7168802584684134413](https://juejin.cn/post/7168802584684134413)

上面介绍的```SETNX + Exipred + 原子操作 + 唯一值校验删除```的方案在很大程度上已经能够满足使用，但是还有一个可能存在的情况没有解决： 
- 锁过期释放，但业务还没执行完成：假设线程a获取锁成功，一直在执行临界区的代码。100s过去后，它还没执行完。但这时候锁已经过期了，此时线程b又请求过来。显然线程b就可以获得锁成功，也开始执行临界区的代码。那么问题就来了，临界区的业务代码都不是严格串行执行。   

针对这种问题，Redisson框架做了一些额外的优化，在加锁的同时开启一个定时守护线程，每隔一段时间检查锁是否还存在，存在则对锁的过期时间延长，防止锁过期提前释放。  

Redisson主要原理及结构如下：  
- 基于Redis命令的实现： Redisson利用了Redis的单线程特性和原子操作的特点。它通过调用Redis的SETNX命令来尝试获取锁，当锁不存在时，才能获取到锁。
- 心跳续约(看门狗)机制： 为防止业务逻辑还没执行完锁就到期的问题，Redisson在获取锁之后会启动一个定时任务来周期性地续约锁的有效时间。
- 实现可重入锁： Redisson支持可重入锁，保证同一线程在持有锁的情况下能够多次获取锁，而不会因为自己已经持有锁而被阻塞。  
- 分布式锁释放的安全性保证： Redisson通过Lua脚本来释放锁，保证了释放锁的原子性。使用Lua脚本可以保证释放锁的操作是原子的，避免了在执行释放锁逻辑时出现的并发问题。  

![](/img/Redis分布式锁/redisson.png)   
 
> 部分核心逻辑如下： 
> - 加锁 
>     - 每次加锁都有一个加锁等待时间 
>     - 如果加锁成功，直接返回true
>     - 如果加锁失败，则订阅锁释放消息，在加锁等待时间内监听锁释放消息，如果一直没有监听到，则取消订阅并返回false 
>     - 如果在等待时间内监听到锁释放消息，则进入一个不断重试获取锁的循环  
> - 续期机制  
>     - 只有在加锁时没有设置过期时间时才会启用Watch Dog机制  
>     - Watch Dog启动守护线程，守护线程轮询周期为：internalLockLeaseTime/3。internalLockLeaseTime的默认值由lockWatchdogTimeout来配置。默认值为30秒。也就是说默认情况下，守护线程每10秒检查续期
>
>

### 4. RedLock  
Redisson解决了锁超时续期自动释放的问题，但是还有一种极端的情况没有解决：  
- 客户端A尝试在Redis Master节点上锁，客户端A成功获得锁的瞬间，锁数据还没有同步至Slave节点。这时Master挂了，于是发生主从切换，其它客户端连接到Slave节点尝试抢占锁，由于Slave没有客户端A的上锁信息。自然又会有一个新的客户端B抢到锁，此时就会出现两个客户端同时拥有分布式锁的奇葩现像。  

鉴于此，Redis作者提出一种更高级的RedLock算法，它需要部署 N （N >= 2n+1）个独立的 Redis master实例，且实例之间没有任何的联系。也就是说，只要一半以上的 Redis 实例加锁成功，那么 Redlock 依然可以正常运行。  
 
![](/img/Redis分布式锁/redlock.png)  

假设我们有 5 个 Redis 实例，当我们对 order1 这个订单加锁时，先记录当前时间用于统计加锁过程花费的时间，然后依次让 5 个 Redis 实例执行 SET order1 token NX EX 60 命令，最后统计**加锁成功的实例数量以及加锁过程耗费的时间**  
- 统计个数: 当超过一半的加锁成功，认为是成功   
- 统计时间: 避免整体的加锁时长已经超过锁本身的有效时间，比如锁的过期时间是3s，但是加锁过程耗费了4s，肯定是认为加锁失败的。  

解锁过程相对简单，只需要5个实例依次删除redis key即可。   


Redlock在Redisson中也有对应的实现，不过在最新版的Redisson中，Redlock已经不再建议使用.因为现在加锁操作实现，可以等所有从节点数据同步了才算加锁成功。这样的话就可以保证主从切换锁不会丢失了。  
```
8.4. RedLock
This object is deprecated. Use RLock or RFencedLock instead.
```    
可以通过Redis的Wait命令实现主从同步  
>Redis WAIT 命令用来阻塞当前客户端，直到所有先前的写入命令成功传输并且至少由指定数量的从节点复制完成。如果执行超过超时时间（以毫秒为单位），则即使尚未完成指定数量的从结点复制，该命令也会返回。



 