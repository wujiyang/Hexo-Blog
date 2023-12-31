---
title: Golang限流器
date: 2024-01-07 22:35:14
index_img: /img/Golang限流/0.png
tags:
    - Golang
    - 限流器
categories:
    - Golang
---

在高并发业务场景下，保护系统时，常用的"三板斧"有："熔断、降级和限流"。本文主要记录golang中常用的限流算法的实现方式。这里所说的限流并非是网关层面或者服务调度之间的限流，而是业务代码中的逻辑限流。

<!-- more -->   

限流算法常用的几种实现方式有如下四种：
- 计数器
- 滑动窗口
- 漏桶
- 令牌桶

## 1. 计数器 
计数器是一种最简单限流算法，其原理就是：**在一段时间间隔内，对请求进行计数，与阀值进行比较判断是否需要限流，一旦到了时间临界点，将计数器清零。**

- 可以在程序中设置一个变量 count，当过来一个请求我就将这个数 +1，同时记录请求时间。
- 当下一个请求来的时候判断 count 的计数值是否超过设定的频次，以及当前请求的时间和第一次请求时间是否在 1 分钟内。
- 如果在 1 分钟内并且超过设定的频次则证明请求过多，后面的请求就拒绝掉。
- 如果该请求与第一个请求的间隔时间大于计数周期，且 count 值还在限流范围内，就重置 count。  

![](/img/Golang限流/1.png)

### 1.1 简单实现
``` go
type LimitRate struct {
   rate  int           //阀值
   begin time.Time     //计数开始时间
   cycle time.Duration //计数周期
   count int           //收到的请求数
   lock  sync.Mutex    //锁
}

func (limit *LimitRate) Allow() bool {
   limit.lock.Lock()
   defer limit.lock.Unlock()

   // 判断收到请求数是否达到阀值
   if limit.count == limit.rate-1 {
      now := time.Now()
      // 达到阀值后，判断是否是请求周期内
      if now.Sub(limit.begin) >= limit.cycle {
         limit.Reset(now)
         return true
      }
      return false
   } else {
      limit.count++
      return true
   }
}

func (limit *LimitRate) Set(rate int, cycle time.Duration) {
   limit.rate = rate
   limit.begin = time.Now()
   limit.cycle = cycle
   limit.count = 0
}

func (limit *LimitRate) Reset(begin time.Time) {
   limit.begin = begin
   limit.count = 0
}
```  
### 1.2 缺点
计数器算法存在“时间临界点”缺陷。比如每一分钟限制200个请求，可以在00:00:00-00:00:58秒里面都没有请求，在00:00:59瞬间发送200个请求，这个对于计数器算法来是允许的，然后在00:01:00再次发送200个请求，意味着在短短1s内发送了400个请求，如果量更大呢，系统可能会承受不住瞬间流量，导致系统崩溃。  
![](/img/Golang限流/2.png)

## 2. 滑动窗口  
### 2.1 原理

滑动窗口算法将一个大的时间窗口分成多个小窗口，每次大窗口向后滑动一个小窗口，并保证大的窗口内流量不会超出最大值，这种实现比固定窗口的流量曲线更加平滑。

普通时间窗口有一个问题，比如窗口期内请求的上限是200，假设有200个请求集中在前1s的后100ms，200个请求集中在后1s的前100ms，其实在这200ms内就已经请求超限了，但是由于时间窗每经过1s就会重置计数，就无法识别到这种请求超限。

![](/img/Golang限流/3.png)
对于滑动时间窗口，我们可以把1ms的时间窗口划分成一些小窗口，上图中我们用红色的虚线代表一个时间窗口（一分钟），每个时间窗口有 6 个格子，每个格子是 10 秒钟。每过 10 秒钟时间窗口向右移动一格，可以看红色箭头的方向。我们为每个格子都设置一个独立的计数器 Counter，假如一个请求在 0:45 访问了那么我们将第五个格子的计数器 +1（也是就是 0:40~0:50），在判断限流的时候需要把所有格子的计数加起来和设定的频次进行比较即可。   


当用户在0:59 秒钟发送了 200 个请求就会被第六个格子的计数器记录 +200，当下一秒的时候时间窗口向右移动了一个，此时计数器已经记录了该用户发送的 200 个请求，所以再发送的话就会触发限流，则拒绝新的请求。 

### 2.2 缺点
滑动窗口算法是计数器算法的一种改进[计数器就是只有一个格子的滑动窗口]，但从根本上并没有真正解决固定窗口算法的临界突发流量问题。想让限流做的更精确只能划分更多的格子。
## 漏桶算法

## 令牌桶算法