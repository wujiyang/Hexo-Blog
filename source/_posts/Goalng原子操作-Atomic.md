---
title: Golang原子操作-Atomic
date: 2024-11-23 18:52:16
index_img: /img/Golang原子操作-Atomic/0.png
tags:
    - Mutex
    - Golang
categories:
    - Golang
		- 互斥锁与原子操作
---

一个或者多个操作在 CPU 执行的过程中不被中断的特性，称为原子性（atomicity） 。这些操作对外表现成一个不可分割的整体，他们要么都执行，要么都不执行，外界不会看到他们只执行到一半的状态。
<!-- more -->  

## 1. 简介
一个或者多个操作在 CPU 执行的过程中不被中断的特性，称为原子性（atomicity） 。这些操作对外表现成一个不可分割的整体，他们要么都执行，要么都不执行，外界不会看到他们只执行到一半的状态。  

>📌 CPU执行一系列操作时不可能不发生中断，但如果我们在执行多个操作时，能让他们的中间状态对外不可见，那我们就可以宣称他们拥有了"不可分割”的原子性。 

## 2. 基本介绍
### 2.1 互斥锁与原子操作
原子操作和互斥锁均可用于在并发环境中保护共享资源，不过它们在应用场景、实现机制、性能等方面存在一定的差异： 
![](/img/Golang原子操作-Atomic/1.png)  
对一个变量更新的保护，原子操作通常更有效率，并且能利用计算机多核的优势
```go
func mutexAdd() {
    var a int32 = 0
    var wg sync.WaitGroup
    var mu sync.Mutex
    start := time.Now()
    for i := 0; i < 10000000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            a += 1
            mu.Unlock()
        }()
    }
    wg.Wait()
    timeSpends := time.Since(start).Nanoseconds()
    fmt.Printf("use mutex a is %d, spend time: %v\n", a, timeSpends)
}
```
```go
func AtomicAdd() {
    var a int32 = 0
    var wg sync.WaitGroup
    
    start := time.Now()
    for i := 0; i < 10000000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt32(&a, 1)
        }()
    }
    
    
    wg.Wait()
    timeSpends := time.Since(start).Nanoseconds()
    fmt.Printf("use atomic a is %d, spend time: %v\n", atomic.LoadInt32(&a), timeSpends)
}
```
### 2.2 原子能力介绍
Go语言通过内置包sync/atomic提供了对原子操作的支持，其提供的原子操作有以下5类，其中每种类别支持的数据类型有6种：Int32、Int64、Uint32、Uint64、Uintptr、Pointer  他们可以随意组合：
- 增加 (Add)：atomic.AddInt32(addr *int32, delta int32)
- 加载（Load）：atomic.LoadInt32(addr *int32)
- 存储（Store）：atomic.LoadInt32(addr *int32)
- 交换（Swap）：atomic.SwapInt32(addr *int32, new int32)
- 比较并交换（CompareAndSwap）: atomic.CompareAndSwapInt32(addr *int32, old int32, new int32)  

>📌 注意：
>- 所有原子操作方法的被操作数形参必须是指针类型，通过指针变量可以获取被操作数在内存中的地址，从而施加特殊的CPU指令，确保同一时间只有一个goroutine能够进行操作  

详细支持的原子操作有：
```go
// TSL Test-and-Set-Lock，对某个存储器位置写值并返回旧值
//
// old = *addr
// *addr = new
// return old
func SwapInt32(addr *int32, new int32) (old int32)
func SwapInt64(addr *int64, new int64) (old int64)
func SwapUint32(addr *uint32, new uint32) (old uint32)
func SwapUint64(addr *uint64, new uint64) (old uint64)
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)

// FAA Fetch-and- Add，对某个存储器位置加值并返回新值
//
// *addr += delta
// return *addr
func AddInt32(addr *int32, delta int32) (new int32)
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddInt64(addr *int64, delta int64) (new int64)
func AddUint64(addr *uint64, delta uint64) (new uint64)
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)

// CAS Compare-and-Swap，判断某个存储器位置的值是否与指定值相等，如果相等则替换为新的值
// CAS操作修改共享变量时候不需要对共享变量加锁，而是通过类似乐观锁的方式进行检查，
// 本质还是不断的占用CPU 资源换取加锁带来的开销（比如上下文切换开销）。
//
//  if *addr == old {
//      *addr = new
//      return true
//  }
//
// return false
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)

// Read
func LoadInt32(addr *int32) (val int32)
func LoadInt64(addr *int64) (val int64)
func LoadUint32(addr *uint32) (val uint32)
func LoadUint64(addr *uint64) (val uint64)
func LoadUintptr(addr *uintptr) (val uintptr)
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)

// Write
func StoreInt32(addr *int32, val int32)
func StoreInt64(addr *int64, val int64)
func StoreUint32(addr *uint32, val uint32)
func StoreUint64(addr *uint64, val uint64)
func StoreUintptr(addr *uintptr, val uintptr)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
```
### 2.3 Atomic.Value
sync/atomic 的原子读写只提供了 int32、int64、uint32、uint64、uintptr 和 unsafe.Pointer 这几个数据类型。如果是其它数据类型的话就需要使用 atomic.Value 来实现原子读写了
#### 2.3.1 简介
```go
type pair struct {
    x, y int
}

func main() {
    p := pair{1, 2}
    var v atomic.Value

    v.Store(p)
    fmt.Println(v.Load().(pair).x)
    fmt.Println(v.Load().(pair).y)
}
```
atomic.Value在底层的实现中是一个interface，且是一个efaceWords类型的interface。从注释中可以看到，Value的零值是nil，且使用后不允许被拷贝。
Value写入值后，typ存储数据类型，data存储数据值
```go
// A Value provides an atomic load and store of a consistently typed value.
// The zero value for a Value returns nil from Load.
// Once Store has been called, a Value must not be copied.
//
// A Value must not be copied after first use.
type Value struct {
    v any
}

// efaceWords is interface{} internal representation.
type efaceWords struct {
    typ  unsafe.Pointer
    data unsafe.Pointer
}
```
Value的核心逻辑是Store和Load，分别介绍如下
#### 2.3.2 Store
```go
// Store 将v的值设置为val，同一个Value只能存储一个类型，而且不能存储nil，否则会panic
func (v *Value) Store(val any) {
    // 数据校验，存储的值不能为nil
    if val == nil {
        panic("sync/atomic: store of nil value into Value")
    }
    // 获取v和val的指针
    vp := (*efaceWords)(unsafe.Pointer(v))
    vlp := (*efaceWords)(unsafe.Pointer(&val))
    for {
        // 判断是否是首次存储
        typ := LoadPointer(&vp.typ)
        if typ == nil {
            // 首次store时会调用runtime_procPin禁止当前P被抢占，保证一次Store的完整性
            runtime_procPin()
            // 使用CAS抢占乐观锁，将typ修改为中间值，表示正在进行首次存储，如果抢锁失败，直接跳出并进行下一轮操作
            if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(&firstStoreInProgress)) {
                runtime_procUnpin()
                continue
            }
            // 抢锁成功后，存储data和typ
            StorePointer(&vp.data, vlp.data)
            StorePointer(&vp.typ, vlp.typ)
            runtime_procUnpin()
            return
        }
        // 如果不是首次存储，需要看下首次存储是不是还在进行中，如果在进行中，就到等待首次存储结束
        if typ == unsafe.Pointer(&firstStoreInProgress) {
            continue
        }
        // 如果首次存储已经结束，校验新存储的数据和第一次的类型是否一致。
        // 如果不一致直接报错，如果一致则完成数据存储
        if typ != vlp.typ {
            panic("sync/atomic: store of inconsistently typed value into Value")
        }
        StorePointer(&vp.data, vlp.data)
        return
    }
}
```
梳理下流程：
1、首先判断类型如果为nil直接panic；
2、然后通过有个for循环来连续判断是否可以进行值的写入；
3、如果是typ == nil表示是第一次写入,然后给type设置一个标识位，来表示有goroutine正在写入；
4、然后写入值，退出；
5、如果type不为nil，但是等于标识位，表示有正在写入的goroutine，然后继续循环；
6、最后type不为nil，并且不等于标识位，并且和value里面的type类型一样，写入内容，然后退出。
注意：其中使用了runtime_procPin()方法，它可以将一个goroutine死死占用当前使用的P(P-M-G中的processor)，不允许其它goroutine/M抢占,这样就能保证存储顺利完成，不必担心竞争的问题。释放pin的方法是runtime_procUnpin。
![](/img/Golang原子操作-Atomic/2.png)    

#### 2.3.3 Load
理解了Store的逻辑后，Load的逻辑就比较简单了
```go
// Load 返回最近一次设置的值
func (v *Value) Load() (val any) {
    // 首先获取v的指标和类型
    vp := (*efaceWords)(unsafe.Pointer(v))
    typ := LoadPointer(&vp.typ)
    // 如果类型为空或者正在进行首次存储，说明没有存储数据，返回nil
    if typ == nil || typ == unsafe.Pointer(&firstStoreInProgress) {
        // First store not yet completed.
        return nil
    }
    // 获取data，并保存
    data := LoadPointer(&vp.data)
    vlp := (*efaceWords)(unsafe.Pointer(&val))
    vlp.typ = typ
    vlp.data = data
    return
}
```
>📌 使用Value类型时需要注意以下事项：
>    - Value不能用来存储nil值，否则会panic
>    - 一个Value变量不能存储不同类型的值，存储的类型只能是第一个存储值的类型。否则会panic
>    - 尽量不要使用Value存储引用类型的值。修改引用类型的值也会同步修改Value中存储的值  

## 3 简易应用
### 3.1 原子性增加值
```go
func main() {
    var count int32
    var wg sync.WaitGroup

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            atomic.AddInt32(&count, 1)            // 原子性增加值
            fmt.Println(atomic.LoadInt32(&count)) // 原子性加载

            wg.Done()
        }()
    }
    wg.Wait()
    fmt.Println("count: ", count)
}
```
### 3.2 简易自旋锁
存满10000后全部取出
```go
var (
    balance int32 = 1000
    wg      sync.WaitGroup
)

// 存钱
func deposit(value int32) {
    for {

        fmt.Printf("余额: %d\n", balance)
        atomic.AddInt32(&balance, value)
        fmt.Printf("存 %d 后的余额: %d\n", value, balance)
        if balance == 10000 {
            break
        }
        time.Sleep(time.Millisecond * 500)
    }
    wg.Done()
}

// 取钱
func withdrawAll(value int32) {
    defer wg.Done()

    for {
        if atomic.CompareAndSwapInt32(&balance, value, 0) {
            break
        }
        time.Sleep(time.Millisecond * 500)
    }
    fmt.Printf("余额: %d\n", value)
    fmt.Printf("取 %d 后的余额: %d\n", value, balance)
}

func main() {
    wg.Add(2)
    go deposit(1000) // 每次存1000
    go withdrawAll(10000)
    wg.Wait()

    fmt.Printf("当前余额: %d\n", balance)
}
```
### 3.3 无符号整数减法操作
对于uint32和uint64类型，Add方法第二个参数只能接受相应的无符号整数，atomic包没有提供减法SubstractT操作
```go
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddUint64(addr *uint64, delta uint64) (new uint64)
```
对于无符号整数V，我们可以传递-V给AddT方法第二个参数就可以实现减法操作。
```go
func main() {
    var i uint64 = 100
    var j uint64 = 10

    var k = 5
    atomic.AddUint64(&i, -j)
    println(i)

    atomic.AddUint64(&i, -uint64(k))
    println(i)

    // 下面这种操作是不可以的，会报错：constant -5 overflows uint64
    // atomic.AddUint64(&i, -uint64(5))
}
```
## 4. 参考资料
- https://juejin.cn/post/7010590496204521485 
- https://golang-notes.readthedocs.io/en/latest/golang/golang-concurrent-synchronization-for-atomic-operation.html
- https://juejin.cn/post/6907091130039894023
- https://boilingfrog.github.io/2021/03/04/go%E4%B8%AD%E7%9A%84atomic/
- https://iswxw.blog.csdn.net/article/details/127827602