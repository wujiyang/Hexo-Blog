---
title: Golang条件变量-sync.Cond
date: 2024-12-19 20:21:21
index_img: /img/Golang条件变量-Cond/1.png
tags:
    - Cond
    - Golang
categories:
    - Golang
		- 互斥锁与原子操作
---

sync.Cond 条件变量用来协调想要访问共享资源的多个 goroutine，当共享资源的状态发生变化的时候，它可以用来通知被互斥锁阻塞的 goroutine。

<!-- more -->  
    
## 1. 基本概念
sync.Cond是Go语言标准库中的一个类型，代表条件变量。它是一种同步机制，用来协调多个goroutine之间的数据同步，当共享资源的状态发生变化时，可以通过条件变量来通知或者等待goroutine，以便他们可以在特定的条件下等待或继续执行。  

### 1.1 适用场景
条件变量的主要作用是在多线程或多进程环境中，实现线程或进程之间的协作和同步，避免不必要的忙碌等待和资源浪费。它可以用于解决生产者 - 消费者问题、线程间的任务分配等多种同步场景。  

比如，有一种典型的生产者消费者场景。我们有一个生产者 goroutine 来生产数据， 但是这个数据生产的流程比较耗时，如果消费者不断的轮询数据是否生产完成会导致忙碌等待和资源浪费。这时不如消费者先阻塞，使用条件变量来实现生产者和消费者之间的同步， 当数据准备好之后，就可以通过条件变量来通知所有等待的消费者 goroutine 去读取数据。  

### 1.2 类型对比
- 条件变量 VS 互斥锁  

条件变量通常与互斥锁（Mutex）结合使用，用于解决线程或进程之间的等待和通知问题。但它们的作用和使用方式又有所不同。  

互斥锁（Mutex）主要用于保护共享资源，确保在同一时刻只有一个线程或进程能够访问该资源。互斥锁通过加锁和解锁来实现对共享资源的独占访问，防止多个线程或进程同时修改共享资源导致的数据不一致或竞态条件。  

条件变量（Condition Variable）则用于线程或进程之间的等待和通知。当一个线程或进程需要等待某个条件满足时，它可以使用条件变量进行等待。其他线程或进程可以在条件满足时通过条件变量通知等待的线程或进程。条件变量通常与互斥锁结合使用，在等待条件变量之前需要先获取互斥锁，以保护条件的判断和修改。  

具体来说，互斥锁用于解决资源竞争问题，而条件变量用于解决线程或进程之间的协作问题。互斥锁关注的是对资源的访问控制，而条件变量关注的是线程或进程之间的状态变化和通知机制。
例如，在生产者 - 消费者问题中，互斥锁用于保护共享的缓冲区，确保生产者和消费者在访问缓冲区时不会发生冲突。而条件变量用于实现生产者和消费者之间的同步，当缓冲区为空时，消费者等待生产者生产数据并通知；当缓冲区已满时，生产者等待消费者取出数据并通知。总的来说，互斥锁和条件变量相互配合，共同实现多线程或多进程的同步和协作。  

- 条件变量 vs 信号量  

信号量也和条件变量类似，用于多线程或多进程同步和通信的机制，但它们有一些区别：  

条件变量主要用于线程或进程之间的等待和通知。当一个线程或进程需要等待某个条件满足时，它可以使用条件变量进行等待，其他线程或进程可以在条件满足时通过条件变量通知等待的线程或进程。条件变量通常与互斥锁结合使用，以保护条件的判断和修改。  

信号量主要用于控制对共享资源的访问数量。它是一个整数计数器，表示可用资源的数量。线程或进程可以通过对信号量进行 P 操作（减 1）来申请资源，如果信号量的值小于等于 0，则线程或进程会被阻塞；通过 V 操作（加 1）来释放资源，唤醒一个或多个等待的线程或进程。  

条件变量适用于需要线程或进程之间进行等待和通知的场景，例如生产者 - 消费者问题、线程间的任务分配等。信号量适用于控制同时访问某个共享资源的线程或进程数量，例如限制同时访问数据库连接的数量、控制并发任务的执行数量等。  

总的来说，条件变量更侧重于线程或进程之间的协作和状态同步，而信号量更侧重于资源的访问控制和并发数量的限制。在实际应用中，根据具体的需求选择合适的同步机制。
- 互斥锁用于解决资源竞争问题
- 条件变量侧重于线程和进程之间的协作和状态同步
- 信号量更侧重于资源的访问控制和并发数量的限制
## 2.简易操作
sync.Cond 的基本用法非常简单，我们只需要通过 sync.NewCond 方法来创建一个 Cond 实例， 然后通过 Wait 方法来等待条件满足，通过 Signal 或者 Broadcast 方法来通知所有等待的 goroutine 去重新获取共享资源。举例如下：一个单生产者多消费者的场景。  

```go
var mutex = sync.Mutex{}
var cond = sync.NewCond(&mutex)

var queue []int

func producer() {
    i := 0
    for {
        mutex.Lock()
        i++
        queue = append(queue, i)
        mutex.Unlock()

        cond.Signal()
        time.Sleep(1 * time.Second)
    }
}

func consumer(consumerName string) {
    for {
        mutex.Lock()
        for len(queue) == 0 {
            cond.Wait()
        }

        fmt.Println(consumerName, queue[0])
        queue = queue[1:]
        mutex.Unlock()
    }
}

func main() {
    // 开启一个 producer
    go producer()

    // 开启两个 consumer
    go consumer("consumer-1")
    go consumer("consumer-2")

    for {
        time.Sleep(1 * time.Minute)
    }
}
```

在以上代码中，有一个 producer 的 goroutine 将数据写入到 queue 中，有两个 consumer 的 goroutine 负责从队列中消费数据。而 producer 和 consumer 对 queue 的读写操作都由sync.Mutex进行并发安全的保护。其中 consumer 需要等待 queue 不为空时才能进行消费，因此 consumer 对于 queue 不为空这一条件的等待和唤醒，就可以用到 sync.Cond。
回过头来重新看下 sync.Cond 接口的用法:
1. sync.NewCond(l Locker): 新建一个 sync.Cond 变量。该函数需要一个 Locker 作为必填参数，这是因为在 cond.Wait() 中底层会涉及到 Locker 的锁操作。
2. cond.Wait(): 等待被唤醒。唤醒期间会解锁并切走 goroutine。
3. cond.Signal(): 只唤醒一个最先等待的 goroutine。
4. cond.Broadcast()：唤醒全部等待的 goroutine。  

## 3. 实现原理
### 3.1 数据结构
sync.Cond的定义如下，提供了Wait ,Singal,Broadcast以及NewCond方法
```go
type Cond struct {
    // noCopy 是一个特殊的字段，用于在静态代码分析工具（如 go vet）中检查不安全的复制   
    noCopy noCopy   
    // L 是一个检查条件或者处理过程中需要持有的锁  
    L Locker   
    
    // 通知队列
    notify  notifyList
    // 检查器，校验Cond是否被复制 
    checker copyChecker
}

func NewCond(l Locker) *Cond {}
func (c *Cond) Wait() {}
func (c *Cond) Signal() {}
func (c *Cond) Broadcast() {}
```
当前结构体的核心内容是一个通知队列notifyList，他是一个链表，保存了所有处于等待状态的协程。通知队列定义如下:
```go
type notifyList struct {
    wait   uint32   
    notify uint32   
    lock   uintptr // key field of the mutex   
    head   unsafe.Pointer   
    tail   unsafe.Pointer
}
```
notifyList包含三类属性：
- wait和notify：他们都是数字，分别代表着写一个等待者的编号和下一个通知者的编号。
- lock：底层是一个mutex类型的指针，用来保护notifyList。
- head和tail：底层是一个sudog类型的指针，用来记录阻塞的goroutine链表的头结点和尾节点。
notifyList 提供了一系列函数，来实现notifyList 读写操作，sync.Cond 也是调用这些方法来实现goroutine的阻塞与唤醒：
- notifyListAdd 将 waiter 的编号加 1。
- notifyListWait 将当前的 goroutine 加入到 notifyList 中。（也就是将当前协程挂起）
- notifyListNotifyOne 将 notifyList 中的第一个 goroutine 唤醒。
- notifyListNotifyAll 将 notifyList 中的所有 goroutine 唤醒。
- notifyListCheck 方法检查 notifyList 的大小是否正确。  

### 3.2 Wait 方法实现
Wait方法首先调用runtime_notifyListAd方法，将自己加入到等待队列中，然后释放锁，等待其他协程的唤醒。
```go
func (c *Cond) Wait() {
    // 检查是否被复制
    c.checker.check()   
    // 将自己放到等待队列中
    // 更新notifyList中需要等待的waiter的数量，返回当前需要插入notifyList的编号
    t := runtime_notifyListAdd(&c.notify)   
    // 释放锁   
    c.L.Unlock()   
    // 挂起当前g，等待被唤醒   
    runtime_notifyListWait(&c.notify, t)   
    // 重新获取锁   
    c.L.Lock()
}
``` 

### 3.3 Signal 方法实现
Singal方法调用runtime_notifyListNotifyOne唤醒等待队列中的一个协程。
```go
func (c *Cond) Signal() {
    // 检查 sync.Cond是否被复制
    c.checker.check()
   // 唤醒等待队列中的一个协程
   runtime_notifyListNotifyOne(&c.notify)
}
```
![图片](/img/Golang条件变量-Cond/2.png) 

### 3.4 Broadcast 方法实现
Broadcast方法调用runtime_notifyListNotifyAll唤醒所有处于等待状态的协程。
```go
func (c *Cond) Broadcast() {
    // 检查 sync.Cond是否被复制
    c.checker.check()
   // 唤醒等待队列中所有的协程
   runtime_notifyListNotifyAll(&c.notify)
}
```
### 3.5 copyChecker 方法实现
在调用 Wait、Signal、Broadcast 方法的时候都会先调用 copyChecker.check 方法，目的是检查 Cond 实例有没有被复制。
```go
// copyChecker holds back pointer to itself to detect object copying.
type copyChecker uintptr

func (c *copyChecker) check() {
    if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
        !atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
        uintptr(*c) != uintptr(unsafe.Pointer(c)) {
        panic("sync.Cond is copied")
    }
}
```
具体这段逻辑未理解，只知道这个是用来校验sync.Cond是否被复制
## 4. 应用场景
### 4.1 生产者与消费者
假设有一个生产者协程不断生产产品并放入缓冲区，有多个消费者协程从缓冲区中取出产品进行消费。可以使用sync.Cond来协调生产者和消费者，当缓冲区为空时，消费者协程等待；当缓冲区已满时，生产者协程等待。
```go
package main

import (
    "fmt"
    "sync"
)

type Buffer struct {
    data []int
    size int
    mutex *sync.Mutex
    notEmpty sync.Cond
    notFull  sync.Cond
}

func NewBuffer(size int) *Buffer {
    mutex := sync.Mutex{}
    return &Buffer{
        size: size,
        mutex: &mutex,
        notEmpty: *sync.NewCond(&mutex),
        notFull:  *sync.NewCond(&mutex),
    }
}

func (b *Buffer) Put(item int) {
    b.mutex.Lock()
    for len(b.data) == b.size {
        b.notFull.Wait()
    }
    b.data = append(b.data, item)
    b.notEmpty.Signal()
    b.mutex.Unlock()
}

func (b *Buffer) Get() int {
    b.mutex.Lock()
    for len(b.data) == 0 {
        b.notEmpty.Wait()
    }
    item := b.data[0]
    b.data = b.data[1:]
    b.notFull.Signal()
    b.mutex.Unlock()
    return item
}

func producer(buffer *Buffer) {
    for i := 0; ; i++ {
        buffer.Put(i)
        fmt.Println("Produced:", i)
    }
}

func consumer(buffer *Buffer) {
    for {
        item := buffer.Get()
        fmt.Println("Consumed:", item)
    }
}

func main() {
    buffer := NewBuffer(5)
    go producer(buffer)
    go consumer(buffer)
    go consumer(buffer)
    select {}
}
```
### 4.2 任务队列管理
有一个任务队列，多个工作协程从队列中获取任务进行处理，当队列为空时等待新任务的加入。管理者协程负责向队列中添加任务，并在添加任务后通知等待的工作协程。
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type TaskQueue struct {
    queue    []interface{}
    mutex    *sync.Mutex
    notEmpty sync.Cond
}

func NewTaskQueue() *TaskQueue {
    mutex := &sync.Mutex{}
    return &TaskQueue{
       mutex:    mutex,
       notEmpty: *sync.NewCond(mutex),
    }
}

func (tq *TaskQueue) AddTask(task interface{}) {
    tq.mutex.Lock()
    tq.queue = append(tq.queue, task)
    tq.notEmpty.Signal()
    tq.mutex.Unlock()
}

func (tq *TaskQueue) GetTask() interface{} {
    tq.mutex.Lock()
    for len(tq.queue) == 0 {
       tq.notEmpty.Wait()
    }
    task := tq.queue[0]
    tq.queue = tq.queue[1:]
    tq.mutex.Unlock()
    return task
}

func worker(tq *TaskQueue) {
    for {
       task := tq.GetTask()
       fmt.Println("Processing task:", task)
    }
}

func main() {
    taskQueue := NewTaskQueue()
    for i := 0; i < 3; i++ {
       go worker(taskQueue)
    }
    for j := 0; j < 10; j++ {
       taskQueue.AddTask(j)
    }

    time.Sleep(3 * time.Second)
}
```
## 5. 注意事项
### 5.1 调用Wait要加锁
sync.Cond必须与互斥锁一起使用，以确保对条件变量的访问是线程安全的。
- 从源码的角度看，在 Wait 方法内部会先解锁然后再进行阻塞，如果调用 Wait 方法前没有进行加锁会出现panic。
- 从应用场景来说，通常 Wait 等待的共享资源是并发访问的，加锁的目的是为了保证共享资源的并发安全。
```go
// 官方文档里的一个示例
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```
### 5.2 Wait条件循环判断
在使用cond.Wait时，可能会出现虚假唤醒的情况，即协程在没有收到信号的情况下被唤醒。因此，在等待条件满足的循环中，应该始终检查条件是否真正满足。
```go
// bad case
c.L.Lock()
if !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()

// good case 
c.L.Lock()
for !condition() {
    c.Wait()
}
... make use of condition ...
c.L.Unlock()
```
### 5.3 不要复制sync.Cond
sync.Cond类型包含了一个互斥锁(mutex)和一个notifyList
1. 如果对其进行复制，此时已经处于使用状态的sync.Mutex会在另一个地方在不知情的情况下使用，这会导致不可预料的情况出现。
2. 其次是notifyList也会被拷贝，notifyList保存了等待通知的goroutine列表。如果拷贝了sync.Cond类型的值，此时新的值和原始值都将指向同一个等待通知的goroutine列表。对新的值调用Singal和Broadcast方法将会影响到原始sync.Cond中的等待通知的goroutine,这样子可能会导致重复唤醒问题的出现。
于是，sync.Cond从实现上就禁止了sync.Cond的复制，在编译期就对其进行验证，一旦试图复制时，编译器会直接报错。
### 5.4 Signal和Boardcast操作不需要加锁
## 6. 参考资料
- 深入理解go语言中的sync.Cond： https://juejin.cn/post/7212270321621483578
- 探索go sync.Cond的实现与应用：https://mp.weixin.qq.com/s/zoe-WIGnC7QCTJhBUigZeg
- 深入理解go sync.Cond https://juejin.cn/post/7194704072136966181
- Golang sync.Cond 条件变量源码分析：https://www.cyhone.com/articles/golang-sync-cond/
- Go sync.Cond 条件变量的学习：https://iswxw.blog.csdn.net/article/details/127723413