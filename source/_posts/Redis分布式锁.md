---
title: Redis分布式锁
date: 2023-12-23 18:13:13
index_img: /img/Redis分布式锁/1.png
tags:
    - Redis
    - Lock
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

上面介绍的```SETNX + Exipred + 原子操作 + 唯一值校验删除```的方案在很大程度上已经能够满足使用，但是还有一个可能存在的情况没有解决： 
- 锁过期释放，但业务还没执行完成：假设线程a获取锁成功，一直在执行临界区的代码。100s过去后，它还没执行完。但这时候锁已经过期了，此时线程b又请求过来。显然线程b就可以获得锁成功，也开始执行临界区的代码。那么问题就来了，临界区的业务代码都不是严格串行执行。
### 4. RedLock
