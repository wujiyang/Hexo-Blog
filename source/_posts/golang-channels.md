---
title: Golang Channels 解析
date: 2024-09-08 14:12:24
index_img: /img/golang-channels/00.png
tags:
    - Channel
    - Golang
categories:
    - Golang
		- 并发编程
---

Channel是一个先进先出的管道，它提供了一种同步的机制，确保数据发送和接收之间的顺序以及正确性。通过使用channel，我们可以避免在多个协程之间共享数据时出现的竞争条件和其他并发问题。

<!-- more -->  

## 1. 简介
在Go语言中，一种广为人知的并发编程模型是CSP(Communicating Sequential Processes)模型，提倡通过通信共享内存，而不是通过共享内存来通信。而channel正是这样一种特殊的类型，用于在并发编程中实现不同协程之间的通信和同步。  

Channel是一个先进先出的管道，它提供了一种同步的机制，确保数据发送和接收之间的顺序以及正确性。通过使用channel，我们可以避免在多个协程之间共享数据时出现的竞争条件和其他并发问题。
![](/img/golang-channels/1.png)  
通道类型的值本身是并发安全的，这也是 Go 语言自带的、唯一一个可以满足并发安全性的类型

## 2. 基本特性
在Go中，channel的关键字是chan，通过箭头表示数据流向←，具有以下基本特性：
- Channel是类型相关的，特定类型的Channel只能存放特定类型的数据。
- channel分缓冲型(buffered)和非缓冲型(unbuffered），非缓冲型的size是0，缓冲型的size是缓冲区的大小。
- 缓冲型通道读写经过buffer，非缓冲型通道的数据交互不经过通道，直接通过内存写传递。
- channel分单向和双向，单向的意思是只能读（或写），双向是既能读也能写。
- channel满时“写”会阻塞，channel空时“读”会阻塞。

### 2.1 无缓冲通道
无缓冲的 channel（unbuffered channel），其缓冲区大小则默认为 0。在功能上其接受者会阻塞等待并阻塞应用程序，直至收到通信和接收到数据。
![](/img/golang-channels/2.png)  
无缓冲通道在处理时，必须同时有发送者和接收者，不然就会阻塞。如下的案例中，如果先在main中往ch写数据，运行时就会抛错：fatal error: all goroutines are asleep - deadlock!
```go
func main() {
	ch := make(chan int)
	// 启用goroutine从通道接收值
	go func() {
		ret := <-ch
		fmt.Println("接收成功: ", ret)
	}()

	ch <- 10
	fmt.Println("发送成功")
	time.Sleep(time.Second)
}
// 发送成功
// 接收成功:  10
```
### 2.2 有缓冲通道
![](/img/golang-channels/3.png) 
解决无缓冲通道（阻塞）死锁的问题，就是使用有缓冲的通道。通过缓存的使用，可以尽量避免阻塞，提供应用的性能。只要通道的容量大于零，那么该通道就是有缓冲的通道，通道的容量表示通道中能存放元素的数量。
```go
func main() {
	ch := make(chan int, 1) // 创建一个容量为 1 的有缓冲区的通道

	ch <- 10
	fmt.Println("发送成功")

	ret := <-ch
	fmt.Println("接收成功: ", ret)
}
```
### 2.3 单向通道
有时候我们将通道作为参数在多个函数间传递时，可以根据不同的任务类型对通道进行限制，比如限制通道在函数中只能发送或只能接收。
注意：发送和接收是针对channel而言，比如发送是指向channel发送数据，接收是指从channel接收数据，可以通过箭头的数据流向来标识类型。
```go
ch := make(chan<- int) // 单向发送通道
ch := make(<-chan int) // 单向接收通道
```
如下一个场景中，首先往一个单向发送通道写入数据，然后经中转将数据写入另一个通道，最后再以单向接收通道的形式读出数据
```go
// 单向发送通道，将数据写入channel
func a(in chan<- int) {
	for i := 0; i < 10; i++ {
		in <- i
	}
	close(in)
}

// 单向发送 in 通道， 单向接收 out 通道，将数据从一个channel写入另一个channel
func a2b(in chan<- int, out <-chan int) {
	for i := range out {
		in <- i * i
	}
	close(in)
}

// 单向接收通道，从channel读出数据
func b(out <-chan int) {
	for i := range out {
		fmt.Println(i)
	}
}

func main() {
	in := make(chan int)
	out := make(chan int)
	go a(in)
	go a2b(out, in)
	b(out)
}
// 0 1 4 9 16 25 36 49 64 81
```
### 2.4 Channel的遍历
如何优雅的从通道中获取数据，Go支持以下几种方式：
- `for range`循环
```go
func main() {
	c := make(chan int, 3)
	for i := 0; i < 3; i = i + 1 {
		c <- i
	}
	close(c)

	for i := range c {
		fmt.Println(i)
	}
	fmt.Println("Finished")
}
// 0
// 1
// 2
// Finished
```
需要注意的是，在遍历时如果channel 没有关闭，那么会一直等待下去，出现 deadlock 的错误；如果在遍历时channel已经关闭，那么在遍历完数据后自动退出遍历。也就是说，for range 的遍历方式是阻塞型的遍历方式。
- `for {}`死循环
通过for{} 死循环+通道关闭判断的方式来进行取值
```go
func main() {
	c := make(chan int)
	go func() {
		for i := 0; i < 3; i = i + 1 {
			c <- i
		}
		close(c)
	}()

	for {
		i, ok := <-c // 通道关闭后再取值ok=false
		if !ok {
			break
		}
		fmt.Println(i)
	}
	fmt.Println("Finished")
}
// 0
// 1
// 2
// Finished
```
- `for select {}`操作
select可以支持一组操作处理，通过case语句标识不同的处理场景。如果有多个case满足条件，则go会随机选择其中一个执行，如果没有符合条件的case，则会选择default语句处理。注意：如果没有default语句且没有满足的条件时，select语句会一直阻塞，一般通过select + default 避免阻塞问题.
```go
func fibonacci(c, quit chan int) {
	x, y := 1, 2
	for {
		select {
		case c <- x:
			x, y = y, x+y
		case <-quit:
			fmt.Println("quit")
			return
		}

	}
}
func main() {
	c := make(chan int)
	quit := make(chan int)
	go func() {
		for i := 0; i < 3; i++ {
			fmt.Println(<-c)
		}
		quit <- 0
	}()
	fibonacci(c, quit)
}
```

### 2.5 异常情况
![](/img/golang-channels/4.png) 
#### 2.5.1 发生Panic
- 向一个关闭的 channel 进行写操作
- 关闭一个 nil 的 channel
- 关闭一个已经关闭的 channel
#### 2.5.2 发生阻塞
- 读一个 nil channel 会被阻塞
- 写一个 nil channel 会被阻塞
- 读写缓冲为空或者已经满了且没有挂起的接收者和发送者。
## 3. 常见应用
### 3.1 通知信号
通过ch来发送停止信号，当main协程将准备工作做好之后，只需要往ch中写入数据（或关闭chan)，就能够让新起的协程退出for循环。下面中select语法类似于switch，但不同的是，case后面只能是一个面向channel的操作。当没有命中case时，会执行default，因此可以通过select来进行非阻塞的读或写。
```go
func main() {
	ch := make(chan bool)
	go func() {
		for {
			// do something recursively
			select {
			case <-ch:
				fmt.Println("channel received")
				return
			default:
				fmt.Println("waiting")
			}
		}
	}()
	// do something until it is time to terminate goroutine
	ch <- true // or close(ch)
}
```
### 3.2 定时任务
与 timer 结合，实现定时任务执行
```go
func worker() {
	ticker := time.Tick(1 * time.Second)
	for {
		select {
		case <-ticker:
			fmt.Println("执行 1s 定时任务")
		}
	}
}

func main() {
	go worker()

	time.Sleep(time.Second * 5)
}
```
### 3.3 生产消费解耦
服务启动时，启动 n 个 worker，作为工作协程池，这些协程工作在一个 for {} 无限循环里，从某个 channel 消费工作任务并执行
```go
func main() {
	taskCh := make(chan int, 10)
	go worker(taskCh)

	// 塞任务
	for i := 0; i < 10; i++ {
		taskCh <- i
	}

	// 等待任务执行完成
	time.Sleep(time.Second * 5)
}

func worker(taskCh <-chan int) {
	const N = 5
	// 启动 5 个工作协程
	for i := 0; i < N; i++ {
		go func(id int) {
			for {
				task := <-taskCh
				fmt.Printf("finish task: %d by worker %d\n", task, id)
				time.Sleep(time.Second)
			}
		}(i)
	}
}
```
### 3.4 并发数控制
有时需要定时执行几百个任务，例如每天定时按城市来执行一些离线计算的任务。但是并发数又不能太高，因为任务执行过程依赖第三方的一些资源，对请求的速率有限制。这时就可以通过 channel 来控制并发数。
```go
var limit = make(chan int, 10)

func main() {
	for _, w := range workers {
		limit <- 1 // limit <- 1在此处会限制创建的goroutine的个数
		go func() {
			w()
			<-limit
		}()
	}
}
```
## 4. 实践避坑
### 4.1 死锁
go 语言新手在编译时很容易碰到这个死锁的问题：fatal error: all goroutines are asleep - deadlock! 在操作系统中，「死锁」就是两个线程互相等待，耗在那里，最后程序不得不终止。go 语言中的「死锁」也是类似的，两个 goroutine 互相等待，导致程序耗在那里，无法继续跑下去.
- 只有生产者或者消费者
channel 的生产者和消费者必须成对出现，如果缺乏一个，就会造成死锁，例如：
```go
// 只有生产者，没有消费者
func f1() {
    ch := make(chan int)
    ch <- 1
}

// 只有消费者，没有生产者
func f2() {
    ch := make(chan int)
    <-ch
}
```
- 生产者和消费者在同一个协程
除了需要成对出现，还需要出现在不同的 goroutine 中，例如：
```go
// 同一个 goroutine 中同时出现生产者和消费者
func f3() {
    ch := make(chan int)
    ch <- 1  // 由于消费者还没执行到，这里会一直阻塞住
    <-ch
}
```
- buffered channel 已满，且在同一个goroutine中
buffered channel 会将收到的元素先存在 hchan 结构体的 ringbuffer 中，继而才会发生阻塞。而当发生阻塞时，如果阻塞了主 goroutine ，则也会出现死锁。
所以实际使用中，推荐尽量使用 buffered channel ，使用起来会更安全
### 4.2 内存泄漏
内存泄漏一般都是通过 OOM(Out of Memory) 告警或者发布过程中对内存的观察发现的，服务内存往往都是缓慢上升，直到被系统 OOM 掉清空内存再周而复始。在 go 语言中，错误地使用 channel 会导致 goroutine 泄漏，进而导致内存泄漏。
让 goroutine 泄漏的核心就是：生产者/消费者 所在的 goroutine 已经退出，而其对应的 消费者/生产者 所在的 goroutine 会永远阻塞住，直到进程退出
- 生产者阻塞导致泄漏
使用channel做超时控制时，假设客户端超时为 500ms，而实际请求耗时为 2s，则 select 会走到 timeout 的逻辑，这时 g2 退出，channel ch 没有消费者，会一直在等待状态；如果这是在 server 代码中，这个请求处理完后，g1 就会挂起、发生了泄漏。
```go
func leak1() {
    ch := make(chan int)
    // g1
    go func() {
        time.Sleep(2 * time.Second) // 模拟 io 操作
        ch <- 100                   // 模拟返回结果
    }()

    // g2
    // 阻塞住，直到超时或返回
    select {
    case <-time.After(500 * time.Millisecond):
        fmt.Println("timeout! exit...")
    case result := <-ch:
        fmt.Printf("result: %d\n", result)
    }
}
```
- 消费者阻塞导致泄漏
如果生产者不继续生产，消费者所在的 goroutine 也会阻塞住，不会退出，例如：
```go
func leak2() {
    ch := make(chan int)

    // 消费者 g1
    go func() {
        for result := range ch {
            fmt.Printf("result: %d\n", result)
        }
    }()

    // 生产者 g2
    ch <- 1
    ch <- 2
    time.Sleep(time.Second)  // 模拟耗时
    fmt.Println("main goroutine g2 done...")
}

```
这种情况下，只需要增加 close(ch) 的操作即可，for-range 操作在收到 close 的信号后会退出、goroutine 不再阻塞，能够被回收。
- 如何预防泄漏
预防 goroutine 泄漏的核心就是：创建 goroutine 时就要想清楚它什么时候被回收。具体到执行层面，包括：
    - 当 goroutine 退出时，需要考虑它使用的 channel 有没有可能阻塞对应的生产者、消费者的 goroutine；
    - 尽量使用 buffered channel 使用 buffered channel 能减少阻塞发生、即使疏忽了一些极端情况，也能降低 goroutine 泄漏的概率；
### 4.3 Close Channel
- 是否需要close
除非必须关闭 chan，否则不要主动关闭。关闭 chan 最优雅的方式，就是不要关闭 chan~
当一个 chan 没有 sender 和 receiver 时，即不再被使用时，GC 会在一段时间后标记、清理掉这个 chan。那么什么时候必须关闭 chan 呢？
比较常见的是将 close 作为一种通知机制，尤其是生产者与消费者之间是 1:M 的关系时，通过 close 告诉下游：我收工了，你们别读了
- 谁来关闭channel
chan 关闭的原则:
- Don’t close a channel from the receiver side 不要在消费者端关闭 chan
- Don’t close a channel if the channel has multiple concurrent senders 有多个并发写的生产者时也别关
只要遵循这两条原则，就能避免两种 panic 的场景，即：向 closed chan 发送数据，或者是 close 一个 closed chan。
按照生产者和消费者的关系可以拆解成以下几类情况：
- 一写一读：生产者关闭即可
- 一写多读：生产者关闭即可，关闭时下游全部消费者都能收到通知
- 多写一读：多个生产者之间需要引入一个协调 channel 来处理信号
- 多写多读：与 3 类似，核心思路是引入一个中间层以及使用 try-send 的套路来处理非阻塞的写入.

## 5. 原理解析
### 5.1 底层结构
channel底层是位于go/src/runtime/chan.go文件中的hchan结构体
```go
// runtime package, chan.go
type hchan struct {
    qcount   uint            // 通道里已有元素的数量
    dataqsiz uint            // 通道buffer(环形队列)的容量
    buf      unsafe.Pointer  // 指向buffer内存的指针（只针对有缓冲的 chan）
    elemsize uint16          // 通道中元素大小
    closed   uint32          // 通道是否被关闭的标志。0：未关闭，非0：关闭。
    elemtype *_type          // 通道中元素类型
    sendx    uint            // 通道写时，写到chan buffer中的位置的索引
    recvx    uint            // 通道读时，读的元素在通道中的位置的索引
    recvq    waitq           // 因为通道读而阻塞的协程队列
    sendq    waitq           // 因为通道写而阻塞的协程队列
    lock     mutex           // 互斥锁，保证通道读写线程安全
}

// waitq 是 sudog 的一个双向链表
type waitq struct {
   first *sudog
   last  *sudog
}

// runtime/runtime2.go
// sudog是对go的一个封装
type sudog struct {
    g       *g              // 阻塞的 goroutine
    elem    unsafe.Pointer  // elem用来存储sender发送数据的地址或recver接收变量的地址
    c       *hchan          // 阻塞的 channel
    next     *sudog         
    prev     *sudog
    // other fields...
}

ch <- 1            // 因为“写”而阻塞的goroutine的elem存的是“1”的地址
val, ok := <- ch   // 因为“读”而阻塞的goroutine的elem存的是“val”的地址
```
一个channel的例子
```go
ch := make(chan int, 4)
// 未阻塞
ch <- 1
ch <- 2
ch <- 3
// 发生阻塞
ch <- 4
ch <- 5
ch <- 6
```
![](/img/golang-channels/5.png) 
![](/img/golang-channels/6.png) 

### 5.2 Channel的创建
channel的创建需要使用make关键字，通过make(type, size)来创建各种类型的通道
```go
a := make(chan int)            // unbuffered channel of integers
b := make(chan int, 0)         // unbuffered channel of integers
c := make(chan *os.File, 100)  // buffered channel of pointers to Files

write := make(chan<- int)        // channel of integers only for writing
read := make(<-chan int)         // channel of integers only for reading
```
channel初始化时分为缓冲型和非缓冲型两种，二者最大的区别是缓冲型channel在初始化时会分配ring buf，底层本质上是一个数组。在分配缓冲buf时，如果元素包含指针类型，buf和hchan地址是不连续的，否则就是连续的
```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	// 计算缓冲区需要的总大小（缓冲区大小*元素大小），并判断是否超出最大可分配范围
	// 如果是非缓冲通道或者是通道元素的size为0（比如struct{}），那么mem就是0
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// 缓冲区大小为0，或者channel中元素大小为0（struct{}{}）时，只需分配channel必需的空间即可;不分配缓冲区的内存
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		// buf指针无实际意义
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// channel中元素类型不是指针，分配一片连续内存空间，所需空间等于 缓冲区数组空间 + hchan必需的空间。
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		// // buf指向了缓冲区内存的首地址
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		// 元素包含指针时 分别分配 chan 和 buf 的内存
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	// 其他字段设置
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}
```
整体逻辑较为清晰：
- 首先校验元素类型和缓冲区空间大小，然后创建hchan，分配所需空间。
- 这里有三种情况：
    - 当缓冲区大小为0，或者channel中元素大小为0时，只需分配channel必需的空间即可；
    - 当channel元素类型不是指针时，则只需要为hchan和缓冲区分配一片连续内存空间，空间大小为缓冲区数组空间加上hchan必需的空间；
    - 默认情况，缓冲区包含指针，则需要为hchan和缓冲区分别分配内存。
- 最后更新hchan的其他字段，包括elemsize，elemtype，dataqsiz。
### 5.3 向Channel发送数据
```go
ch := make(chan int)
ch <- 3
```
通道写操作会调用chansend函数
```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// channel为空的场景下，如果是非阻塞，直接返回发送失败，如果是阻塞场景下，将当前协程挂起阻塞
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(chansend))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	// 对于非阻塞队列且channel未关闭、如果无缓冲区且没有等待的接收者或者换冲区已满都直接返回false
	if !block && c.closed == 0 && full(c) {
		return false
	}
	
	// 一般情况下都是阻塞队列，主要看下面的逻辑

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	// 加锁，保证协程并发安全
	lock(&c.lock)
	// 如果channel已关闭，直接panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	
	// 首先判断是否有等待接收者
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		// 有等待接收者的时候分两种情况，无缓冲队列或者是缓冲队列为空
		// 这两种情况都是直接唤醒等待的协程，直接将数据给等待者
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	
	// 如果当前buf中的数量小于容量，说明还有空间，直接把数据写入buf，并更新sendx和qcount等字段
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		// 索引更新及环形队列处理
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	// 如果队列已经满了，则需要构造一个sudog，将发送协程加入发送者队列
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 当前 goroutine 进入发送等待队列
	c.sendq.enqueue(mysg)
	// 挂起协程
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	// 如果协程被唤醒了，说明数据会被读取走；后续执行清理工作并释放sudog结构体
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}
```
主要流程分三种情况：
- 存在等待接收的协程：说明无缓冲或者缓冲为空，直接把数据给接收者
- 没有等待的接收者且buf未满：把数据放入buf末尾
- 没有等待的接收者且buf满了：将发送者加入发送等待队列
![](/img/golang-channels/7.png) 
### 5.4 从Channel接收数据
```go
ch := make(chan int)
value := <- ch // 一个接收值
value, ok := <- ch // 两个接收值
```
- 第3行中，第二个接收值“ok”表示value是否是有效数据，即value的值是否来自sender发送的数据。
📌 
>可以通过“ok”来判断chan是否关闭，即ok=false，则chan关闭。但不能认为ok=true，chan就未关闭，因为在某些chan已经关闭的情况下（如chan的buffer中有数据），“ok”依然是true。
    当ok=true时，chan可能关闭或未关闭；当ok=false时，chan一定是关闭了。  
    
接收数据时，核心调用的是 chanrecv函数
```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// ep：用来接收数据的地址。如果 ep 是 nil，说明忽略了接收值(-)。
	// block表示通道是否是阻塞模式，我们创建的通道都是阻塞模式，下面代码不考虑非阻塞模式
	// received表示接收到的是否是有效数据，即sender发送的数据。
	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}
	// channel为空场景下，如果非阻塞队列直接返回false，如果阻塞队列，直接挂起等待
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	// 非阻塞模式下的快速退出场景，一般无需关注
	if !block && empty(c) {
		if atomic.Load(&c.closed) == 0 {
			return
		}
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}
	
	// 加锁 保证协程并发安全性
	lock(&c.lock)

	
	if c.closed != 0 {
		// 如果channel已经关闭且缓冲区无元素
		if c.qcount == 0 {
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			unlock(&c.lock)
			// 给ep一个零值
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			// 返回（true, false），即接收到值，但不是从channel中接收的有效值
			return true, false
		}
		// The channel has been closed, but the channel's buffer have data.
	} else {
		// Just found waiting sender with not closed.
		// 常规情况1: 有等待发送的队列
        // 有等待发送的队列时，说明有两种情况：非缓冲队列或者缓冲已满；
		// 对于非缓冲情况，直接从sender接收数据
		// 对于缓冲已满情况，从buf队列头部接收数据，并将sender数据加入队列尾部
		if sg := c.sendq.dequeue(); sg != nil {
			// Found a waiting sender. If buffer is size 0, receive value
			// directly from sender. Otherwise, receive from head of queue
			// and add sender's value to the tail of the queue (both map to
			// the same buffer slot because the queue is full).
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			// 接收成功
			return true, true
		}
	}
	
	// 常规情况2: 无等待发送队列，且buf有值；
	// 直接从recvx取数据即可，并更新recvx和qcount的值
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
	// 非阻塞模式，没有数据，直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}
	
	// 阻塞模式，没有数据，则挂起到接收队列中
	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
    // 加入到channel的等待接收队列recvq中
	c.recvq.enqueue(mysg)
	// stack shrinking.
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	// 被唤醒之后执行清理工作并释放sudog结构体
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```
可以看到，在整个读取数据的流程中，和发送数据的流程非常相似，主要分为3种情况
- 存在等待发送的协程：这种场景说明无缓冲通道或者缓冲buf已满
    - 如果无缓冲区，那么直接从sender接收数据； 
    - 如果缓冲区已满，从buf队列的头部接收数据，并把sender的数据加到buf队列的尾部； 
    - 最后调用goready函数将等待发送数据的Goroutine的状态从_Gwaiting置为_Grunnable，等待下一次调度。
- 没有等待发送的协程且buf存在数据，直接从buf中取数据。
- 没有等待发送的协程且无数据，接收者挂起加入接收者队列。
![](/img/golang-channels/8.png) 
### 5.5 关闭Channel
收发完数据后，就是对channel的关闭操作，关闭操作主要执行以下逻辑：
- 遍历recvq和sendq（实际只有recvq或者sendq），取出sudog中挂起的Goroutine加入到glist列表中，并清除sudog上的一些信息和状态。
- 遍历glist列表，为每个Goroutine调用goready函数，将所有Goroutine置为_Grunnable状态，等待调度。
- 当Goroutine被唤醒之后，会继续执行chansend和chanrecv函数中当前Goroutine被唤醒后的剩余逻辑。
```go
func closechan(c *hchan) {
	// 关闭 nil channel，panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		// 关闭closed channel，panic
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, abi.FuncPCABIInternal(closechan))
		racerelease(c.raceaddr())
	}
	// 设置为关闭状态
	c.closed = 1

	var glist gList

	// release all readers
	// 遍历recvq，清除sudog的数据，取出其中处于_Gwaiting状态的Goroutine加入到glist中
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	// 遍历sendq，清除sudog的数据，取出其中处于_Gwaiting状态的Goroutine加入到glist中
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	// 将glist中所有Goroutine的状态置为_Grunnable，等待调度器进行调度
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```
# 参考资料

[Go 并发之channel](https://iswxw.blog.csdn.net/article/details/130785190)

[go channel详解](https://mp.weixin.qq.com/s/rFyMNeZF_cwKkBBJhHlEFQ)

[深入理解 channel](https://mp.weixin.qq.com/s/XEdrrpIkdseFP3ZIdfOgVw)

[The Nature Of Channels In Go](https://www.ardanlabs.com/blog/2014/02/the-nature-of-channels-in-go.html)

[一文带你揭秘Go语言之通道channel](https://mp.weixin.qq.com/s/V7B9q4lZ6K3RjrP8Xd-cXw)

[Channel](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)