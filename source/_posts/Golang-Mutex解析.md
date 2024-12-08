---
title: Golang 互斥锁 sync.Mutex
date: 2024-11-08 14:26:35
index_img: /img/Golang-Mutex解析/0.png
tags:
    - Mutex
    - Golang
categories:
    - Golang
		- 互斥锁与原子操作
---

互斥锁是在并发程序中对共享资源进行访问控制的主要手段，对此 Go 语言提供了简单易用的 Mutex，它是一种用于并发编程中的同步机制，用于确保在同一时刻只有一个 goroutine 可以访问共享资源。

<!-- more -->  

##  1. 背景介绍

### 1.1 基本信息
互斥锁是在并发程序中对共享资源进行访问控制的主要手段，对此 Go 语言提供了简单易用的 Mutex，它是一种用于并发编程中的同步机制，用于确保在同一时刻只有一个 goroutine 可以访问共享资源。
**主要特点**：
- 互斥性：Mutex的核心特性是互斥性，即保证在任何时刻只有一个 goroutine 可以持有锁并访问被保护的共享资源。这有效地防止了多个 goroutine 同时对共享资源进行读写操作而导致的数据竞争和不一致问题。
- 原子性操作：在获取和释放锁的过程中，Mutex的操作是原子性的。这意味着在一个 goroutine 尝试获取锁时，其他 goroutine 不能同时进行获取锁的操作，直到当前持有锁的 goroutine 释放锁。
- 不可重入性：Go语言的Mutex只记录了加锁状态，没有记录锁的所有者，所以不支持可重入，自己加的锁别人也可以打开
### 1.2 为什么要使用Mutex
我们看一个示例，启动 10000 个协程将变量 num 加1，因此肯定会存在并发；如果我们不控制并发，10000 个协程都执行完后，该变量的值很大概率不等于 10000。但是如果使用并发的话，总能得到正确的结果。
```go
func main() {
    num := 0
    threadCount := 10000

    var wg sync.WaitGroup
    wg.Add(threadCount)
    for i := 0; i < threadCount; i++ {
        go func() {
            defer wg.Done()
            num++
        }()
    }

    wg.Wait()        // 等待 10000 个协程都执行完
    fmt.Println(num) // 9388(每次都可能不一样)

}
```
```go
func main() {
    num := 0
    threadCount := 10000

    var mutex sync.Mutex // 互斥锁
    var wg sync.WaitGroup
    wg.Add(threadCount)
    for i := 0; i < threadCount; i++ {
        go func() {
            defer wg.Done()
            mutex.Lock()   // 加锁
            num++          // 临界区
            mutex.Unlock() // 解锁
        }()
    }

    wg.Wait()        // 等待 10000 个协程都执行完
    fmt.Println(num) // 结果为10000
}
```
从上面可以看到，Mutex可以保证并发编程过程中的数据一致性和稳定性，实现并发安全编程。
### 1.3 基本操作
Mutex 是一个结构体，对外提供 Lock()和Unlock()两个方法，分别用来加锁和解锁。
```go
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
    Lock()
    Unlock()
}
```
1. 加锁（Lock）：当一个 goroutine 需要访问被互斥锁保护的共享资源时，它首先需要调用互斥锁的Lock方法。这个方法会阻塞当前线程，直到互斥锁被释放，然后当前线程获得锁并可以继续执行对共享资源的访问操作。
2. 解锁（Unlock）：当一个线程完成对共享资源的访问后，它应该调用互斥锁的Unlock方法来释放锁，以便其他等待的线程可以获得锁并访问共享资源。
## 2. 核心原理
### 2.1 数据结构
Go 语言的 sync.Mutex由两个字段 state 和 sema 组成。其中 state 表示当前互斥锁的状态，而 sema 是用于控制锁状态的信号量。
```go
type Mutex struct {    
    state int32  // 复合字段，不仅能表示锁的状态，还能表示等待锁的协程个数 
    sema  uint32 // 信号量，协程阻塞会等待该信号量，解锁的协程释放信号量从而唤醒等待信号量的协程
}
```
互斥锁的状态比较复杂，最低三位分别表示 mutexLocked、mutexWoken 和 mutexStarving，剩下的位置用来表示当前有多少个 Goroutine 在等待互斥锁的释放
![](/img/Golang-Mutex解析/1.png)    
int32 中的不同位分别表示了不同的状态，默认情况下：互斥锁的所有状态位都是 0，即默认为未锁定状态
- mutexLocked — 表示互斥锁的锁定状态；
- mutexWoken — 表示是否有协程被唤醒；
- mutexStarving — 当前的互斥锁进入饥饿状态；
- waitersCount — 当前互斥锁上等待的 Goroutine 个数
相关常量定义：
```go
// sync/mutex.go 36
const (
    mutexLocked = 1 << iota // 1 0001 含义：用最后一位表示当前对象锁的状态，0-未锁住 1-已锁住
    mutexWoken // 2 0010 含义：用倒数第二位表示是否已经有协程被唤醒 0-未唤醒 1-已有协程唤醒
    mutexStarving // 4 0100 含义：用倒数第三位表示当前Mutex是否为饥饿模式，0为正常模式，1为饥饿模式。
    mutexWaiterShift = iota // 3，从倒数第四位往前的bit位表示在排队等待的goroutine数
    
    starvationThresholdNs = 1e6 // 1ms 切换到饥饿模式的阈值
)
```
### 2.2 Lock
本小节主要介绍Mutex中加锁的基本逻辑及一些核心思想。
#### 2.2.1 常规
Mutex的加锁操作是一个阻塞调用，如果在加锁时已经被使用，则当前协程会阻塞直至获取到锁。
```go
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        if race.Enabled {
            race.Acquire(unsafe.Pointer(m))
        }
        return
    }
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}
```
由上面的逻辑可以看到，一个协程在加锁时，分为Fast path和Slow Path, 如果Fast Path能加锁成功，就直接返回了，否则要进入复杂的Slow Path中，这里不详细介绍每一步的源码，只分析核心的逻辑。
- Fast Path：
    - 如果m.state为0，说明当前锁处于未锁定状态，尝试使用CAS将其变为锁定状态并直接返回
- Slow Path：
    - 判断当前协程能否进入自旋
    - 通过自旋等待互斥锁的释放
    - 计算互斥锁的最新状态（处理完自旋特殊逻辑后，互斥锁会根据上下文计算当前互斥锁最新的状态）
    - 更新互斥锁的状态并尝试获取锁
        - 如果获取到锁直接返回
        - 如果没有获取到锁，会调用runtime_SemacquireMutex使协程陷入休眠并等待被唤醒
        - 唤醒后如果处于正常模式，会设置唤醒和饥饿标记、重置迭代次数并重新执行获取锁的循环；
        - 唤醒后如果处于饥饿模式，当前 Goroutine 会获得互斥锁，如果等待队列中只存在当前 Goroutine，互斥锁还会从饥饿模式中退出；119
#### 2.2.2 自旋
自旋对应 CPU 的 PAUSE 指令，CPU 对该指令什么都不做，相当于空转。对程序而言相当于sleep了很小一段时间，大概 30个时钟周期。  

**自旋条件**  
加锁时runtime 会自动判断是否可以自旋，无限制的自旋将给 CPU 带来巨大压力，自旋必须满足以下条件：
- 互斥锁只有在普通模式才能进入自旋；
- 自旋次数要足够少，通常为 4，即自旋最多 4 次；
- CPU 核数要大于 1，否则自旋没有意义，因为此时不可能有其他协程释放锁；
- 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；
可见自旋的条件是很苛刻的，简单说就是不忙的时候才会启用自旋。  

**自旋优势**   
自旋的优势是更充分地利用 CPU，尽量避免协程切换。因为当前申请加锁的协程拥有 CPU，如果经过短时间的自旋可以获得锁，则当前写成可以继续运行，不必进入阻塞状态。  

**自旋劣势**  
如果在自旋过程中获得锁，那么之前被阻塞的协程就无法获得。如果加锁的协程特别多，每次都通过自旋获取锁，则之前被阻塞的协程将很难获取锁。
#### 2.2.3 Mutex 模式
正常情况下，锁释放唤醒等待协程时，等待协程会和此时新到的协程以及处于自旋状态的协程一起竞争锁，因为需要进行上下文切换，新唤醒协程大概率争抢不过新协程以及自旋的协程，为了解决这种情况下阻塞队列中协程一直获取不到锁的问题，Mutex引入了模式的概念，分为正常模式和饥饿模式。  

**正常模式 Normal**：  
正常模式下，锁的等待者会按照先进先出的顺序获取锁。但是刚被唤起的协程与新创建的协程竞争时，大概率会获取不到锁，为了减少这种情况的出现，一旦 Goroutine 超过 1ms 没有获取到锁，它就会将当前互斥锁切换饥饿模式，防止部分 Goroutine 被饿死。 

> 📌 引入饥饿模式的目的是保证互斥锁的公平性。  

**饥饿模式 Starving**：  
饥饿模式中，互斥锁的所有权直接从解锁的Goroutine交给等待队列最前面的 Goroutine。新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。如果一个 Goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1ms，那么当前的互斥锁就会切换回正常模式。
在Starving模式下，不会启动自旋过程，一旦有协程释放了锁，一定会唤醒协程，被唤醒的协程将成功获取锁，同时会把等待计数减 1。    

>📌  与饥饿模式相比，正常模式下的互斥锁能够提供更好地性能，饥饿模式的能避免 Goroutine 由于陷入等待无法获取锁而造成的长尾延时。 

#### 2.2.4 Woken 状态
Woken 状态用于加锁和解锁过程中的通信。比如，同一时刻，两个协程一个在加锁，一个在解锁，在加锁的协程可能在自旋过程中，此时把 Woken 标记为 1，用于通知解锁协程不必释放信号量，类似知会一下对方，不用释放了，我马上就拿到锁了。
### 2.3 UnLock
相比于加锁，解锁的逻辑简单许多，同样分为Fast path和Slow Path
```go
func (m *Mutex) Unlock() {
    if race.Enabled {
        _ = m.state
        race.Release(unsafe.Pointer(m))
    }

    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        // Outlined slow path to allow inlining the fast path.
        // To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.
        m.unlockSlow(new)
    }
}
```
- Fast Path：
    - 使用原子操作直接将状态移除锁标记，如果新状态为0，就成功释放锁；当前操作下说明没有等待协程也没有被唤醒协程。否则就进入Slow Path
- Slow Path：
    - 校验锁的合法性，避免锁被重复解锁
    - 正常模式：如果互斥锁不存在等待者或者互斥锁的 mutexLocked、mutexStarving、mutexWoken 状态不是全部为 0，那么当前方法可以直接返回，不需要唤醒其他等待者；如果互斥锁存在等待者，会通过runtime_Semrelease唤醒等待者并移交锁的所有权；
    - 饥饿模式：直接释放信号量将锁交给下一个等待者
## 3. 应用场景
### 3.1 保护共享资源
在并发编程中，多个 goroutine 可能同时访问和修改同一个共享资源。为了确保数据的一致性和正确性，需要使用互斥锁来保护共享资源。例如，Map 并发读写时会出现 panic，我们可以使用 Mutex 来保护 Map 防止出现 panic。
```go
type SafeMap struct {
    m    map[string]int
    lock sync.Mutex
}

func (s *SafeMap) Put(key string, value int) {
    s.lock.Lock()
    s.m[key] = value
    s.lock.Unlock()
}

func (s *SafeMap) Get(key string) (int, bool) {
    s.lock.Lock()
    value, ok := s.m[key]
    s.lock.Unlock()
    return value, ok
}
```
### 3.2 同步Goroutine
在某些情况下，需要确保多个 goroutine 按照特定的顺序执行。可以使用互斥锁来实现这种同步。例如，一个 goroutine 需要等待另一个 goroutine 完成某个任务后才能继续执行。可以使用互斥锁来确保两个 goroutine 之间的同步。
```go
var mutex sync.Mutex
var done bool

func worker1() {
    fmt.Println("Worker 1 is working...")
    mutex.Lock()
    done = true
    mutex.Unlock()
}

func worker2() {
    mutex.Lock()
    for !done {
        mutex.Unlock()
        mutex.Lock()
    }
    fmt.Println("Worker 2 can start working...")
    mutex.Unlock()
}

func main() {
    go worker1()
    go worker2()

    time.Sleep(1 * time.Second)
}
```
### 3.3 避免数据竞争
在并发编程中，如果多个 goroutine 同时访问和修改同一个变量，可能会导致数据竞争。使用互斥锁可以避免这种数据竞争。例如，一个 goroutine 正在读取一个变量的值，而另一个 goroutine 正在修改这个变量的值。使用互斥锁可以确保在读取和修改操作之间不会被其他 goroutine 干扰。
```go
var data int
var mutex sync.Mutex

func reader() {
    mutex.Lock()
    fmt.Println("Data:", data)
    mutex.Unlock()
}

func writer() {
    mutex.Lock()
    data++
    mutex.Unlock()
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            reader()
        }()
        wg.Add(1)
        go func() {
            defer wg.Done()
            writer()
        }()
    }
    wg.Wait()
}
```
## 4. 注意事项
mutex使用过程简单，只有基本的加锁和解锁操作，但还是有一些点需要注意：
1. Mutex 是可以在 goroutine A 中加锁，在 goroutine B 中解锁的，但是在实际使用中，尽量保证在同一个 goroutine 中加解锁。多次加锁而未解锁可能导致程序“卡死”（也被称为死锁）。尝试解锁一个未加锁的互斥锁可能引发程序崩溃（panic）。
2. Mutex 的加锁解锁基本都是成对出现，为了解决忘记解锁，可以使用 defer 语句，在加锁后直接 defer mutex.Unlock()；但是如果处理完临界区资源后还有很多耗时操作，为了尽早释放锁，不建议使用 defer，而是在处理完临界区资源后就调用 mutex.Unlock() 尽早释放锁。
3. Mutex 不能复制使用；Mutex 是有状态的，比如我们对一个 Mutex 加锁后，再进行复制操作，会把当前的加锁状态也给复制过去，基于加锁的 Mutex 再加锁肯定不会成功。进行复制操作可能听起来是一个比较低级的错误，但是无意间可能就会犯这种错误。
```go
type Counter struct {
    mutex sync.Mutex
    num   int
}

func AddFunc(c Counter) {
    c.mutex.Lock()
    defer c.mutex.Unlock()
    c.num++
}

func main() {
    var counter Counter
    counter.mutex.Lock()
    defer counter.mutex.Unlock()

    counter.num++
    // Go是值传递，这里复制了 counter
    // 此时 counter.mutex 是加锁状态，在 SomeFunc 无法再次加锁，就会死锁
    AddFunc(counter)
}
```
4. 注意锁的粒度：锁的粒度指的就是临界区的大小，临界区的代码都是串行执行的。如果锁的粒度太大，可能会导致并发度降低；如果锁的粒度太小，可能会增加锁的管理开销。在设计并发程序时，应该根据实际情况选择合适的锁粒度。
5. 避免死锁：在使用互斥锁时，要注意避免死锁的发生。死锁是指两个或多个 goroutine 相互等待对方释放锁，导致程序无法继续执行的情况。为了避免死锁，可以按照固定的顺序获取多个锁，或者使用超时机制来避免无限期地等待锁。
## 5. 源码分析
### 5.1 Lock
Lock()加锁方法分为两部分：
1. 第一部分是 fast path，可以理解为快捷通道，如果当前锁没被占用，直接获得锁返回；否则需要进入 slow path。
2. slow path 判断各种条件去竞争锁，主要逻辑都在此处。
```go
func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // 当锁被占用且不处于饥饿状态且可以执行自旋时，进入自旋逻辑
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 如果当前awoker标志位为false，且等待者不等于0，把自己标记为唤醒的协程
            // 该标记可以告诉Unclock操作不用再唤醒其他协程了(我已经在等锁了，你们不用醒了)
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        // 走到这里说明退出了自旋，当前锁可能 没被占用 或者 锁处于饥饿状态 或者 自旋太多了
        // new 代表当前goroutine基于当前状态接下来要设置的新状态
        new := old
        // 只要不是饥饿状态，就试图获取锁（如果是饥饿状态就乖乖去排队）
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        // 锁被占用 或者 处于饥饿状态下，新增一个等待者
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        
        // 当前goroutine已经进行饥饿状态了，且锁还没释放，需要把Mutex的状态改为饥饿
        if starving && old&mutexLocked != 0 {
            new |= mutexStarving
        }
        // 如果是被唤醒的，把唤醒标志置0，表示外面没有被唤醒的goroutine了
        // 这段的操作是为了下面的抢锁做准备，抢到就获取锁，抢不到就休眠，把唤醒标志设置为0
        if awoke {
            if new&mutexWoken == 0 {
                throw("sync: inconsistent mutex state")
            }
            new &^= mutexWoken
        }
        // 利用CAS尝试将锁的状态切切换成新的，就是上面的一坨操作
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 状态切换成功，且在改状态前 锁未被占用 且 处于正常模式，那么就相当于获取到锁了
            if old&(mutexLocked|mutexStarving) == 0 {
                break // locked the mutex with CAS
            }
            
            // 走到这里说明锁没正常获取到
            // 判断之前是否等待过，之前等待过的，再次排队要放在队首
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            // 休眠等待信号量：之前排过队的老人，放到等待队列队首；新人放到队尾
            runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            
            // 走到这里说明锁被释放，当前协程获取到信号量被唤醒
            // 重新计算当前goroutine的饥饿状态
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            
            // 如果 state 饥饿标记为1，说明当前在饥饿模式，饥饿模式下被唤醒，已经获取到锁了；
            // 饥饿状态下，释放锁没有更新等待者数量和饥饿标记，需要获得锁的goroutine去更新状态
            if old&mutexStarving != 0 {
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                // 加锁，减去一个等待者
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    // 如果当前的 goroutine 非饥饿，或者等待者只有一个（也就是只有当前goroutine，等待队列空了），
                    // 可以取消饥饿状态，进入正常状态
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta)
                break
            }
            // 非饥饿状态被唤醒，标记为awoke，重置自旋迭代次数，继续去进行下一轮的抢锁
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }

    if race.Enabled {
        race.Acquire(unsafe.Pointer(m))
    }
}
```
Mutex本身是多个go协程的共享变量，不同的go协程同时执行该函数，函数内临时变量是go协程自己的状态。每次执行因为状态不同处在函数执行的不同阶段，直到mutex状态改变。
### 5.2 UnLock
Unlock()解锁方法也分为两部分：
1. 第一部分是 fast path，可以理解为快捷通道，直接把锁状态位清除，如果此时系统状态恢复到初始状态，说明没有 goroutine 在抢锁等锁，直接返回，否则进入 slow path；
2. slow path 会根据是否为饥饿状态，做出不一样的反应：
    1. 正常状态：唤醒一个 goroutine 去抢锁，等待者数量减一，并将唤醒状态置为 1；
    2. 饥饿状态：直接唤醒等待队列队首的 goroutine，锁的所有权直接移交（修改等待者数量、是否取消饥饿标记，由唤醒的 goroutine 去处理）。  

```go
func (m *Mutex) unlockSlow(new int32) {
    // 校验是否对已解锁的mutex再次解锁
    if (new+mutexLocked)&mutexLocked == 0 {
        fatal("sync: unlock of unlocked mutex")
    }
    
    // 正常模式，非饥饿
    if new&mutexStarving == 0 {
        old := new
        for {
            // 如果没有等待者，或者已经存在被唤醒的协程，或者已经有协程获得了锁，无需唤醒协程，直接交接返回即可
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // 减去一个等待者，并且将 唤醒标记 置为 1
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false, 1)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿模式下，直接释放信号量，将锁的所有权，交给等待队列的第一个等待者
        runtime_Semrelease(&m.sema, true, 1)
    }
}
```
## 6. 参考资料
- [初见 Go Mutex](https://juejin.cn/post/7146236976399089700)
- [Golang Mutex 原理解析](https://juejin.cn/post/7086756462059323429)
- [Go语言之sync.Mutex 源码分析](https://www.lixueduan.com/posts/go/sync-mutex/)