---
title: Golang Context 解析
date: 2024-08-31 14:09:29
index_img: /img/Golang-Context解析/0.jpg
tags:
    - Context
    - Golang
categories:
    - Golang
		- 并发编程
---

Context 是 Go 语言中用于处理并发操作的一个重要概念。包含 goroutine 的运行状态、环境、现场等信息。Context 主要用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、Key-Value等。

<!-- more -->  

## 1. Context 介绍
Context 是 Go 语言中用于处理并发操作的一个重要概念。Context译作“上下文”，准确说它是 goroutine 的上下文，包含 goroutine 的运行状态、环境、现场等信息。Context 主要用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、Key-Value等。

接口定义如下：
```go
type Context interface {
    Deadline() (deadline time.Time, ok bool) //返回 context 的过期时间；
    Done() <-chan struct{}					 //返回 context 中的 channel；
    Err() error								 //返回错误；
    Value(key interface{}) interface{}		 //返回 context 中的对应 key 的值.
}
```
- `Deadline()`：该方法返回ctx的超时时间，以及是否设置超时时间的标识；如果没有设置超时时间，则ok=false，deadline是一个初始的time.Time值。
- `Done()`：该方法返回一个只读的channel，当context被主动取消或者超时自动取消时，这个channel会被关闭；关闭的channel是可读的，协程可以正常收到关闭信号；如果一个ctx没有设置超时时间，调用该方法会返回nil；经常在select-case语句中使用，判断ctx是否关闭
- `Err()`：该方法返回ctx关闭的原因，如果ctx还没有关闭，就返回nil，如果被关闭了，返回的err也只有两种
```go
var Canceled = errors.New("context canceled")        // 正常取消

var DeadlineExceeded error = deadlineExceededError{} // 超时取消
```
- `Value()`：golang中有一种在协程间传递信息的ctx，使用该方法可以根据key查询map中的value


## 2. Context 派生
Context在设计上实际只定义了接口，凡是实现改接口的结构体都可以称为Context，官方的包中实现了以下几类结构体。 

![](/img/Golang-Context解析/1.png)    

- `emptyCtx`：空结构体，使用Background()函数和TODO()函数创建
- `valueCtx`：值传递的context，使用WithValue()函数创建
- `cancelCtx`：带有取消函数的context，使用WithCancel()函数创建
- `timerCtx`：带有超时时间的context，使用WithDeadline()和WithTimeout()函数创建
## 3. 使用场景
Context在日常使用中，一般多用于超时控制、信号传递、共享变量等场景，下面分别简单介绍一下
### 3.1 取消信号传递
- 在主协程中主动取消ctx，子协程收到取消请求后关闭操作
```go
func worker(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("Worker received cancellation signal: %v\n", ctx.Err())
			return
		default:
			time.Sleep(500 * time.Millisecond)
			fmt.Println("Working...")
		}
	}
}

func main() {
	parentCtx, cancel := context.WithCancel(context.Background())
	go worker(parentCtx)

	time.Sleep(1 * time.Second) // 模拟主程序执行

	cancel() // 取消worker的ctx

	time.Sleep(1 * time.Second)
}
// Working...
// Working...
// Worker received cancellation signal: context canceled
```

### 3.2 超时控制
- 使用带有超时时间的ctx，时间到后，函数自动中断处理
```go
func operation(ctx context.Context) {
	select {
	case <-time.After(3 * time.Second): // Simulate some long operation
		fmt.Println("Operation completed.")
	case <-ctx.Done():
		fmt.Println("Operation canceled due to timeout.")
	}
}

func main() {
	timeoutCtx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	operation(timeoutCtx)
}
// Operation canceled due to timeout.
```

### 3.3 截止时间
- ctx带有截止时间，在deadline之前可以正常工作，deadline后会报错
```go
func operation(ctx context.Context) {
	deadline, ok := ctx.Deadline()
	if ok {
		fmt.Printf("Operation deadline: %s\n", deadline)
	} else {
		fmt.Println("No deadline for the operation.")
	}

	// Simulate some operation
	time.Sleep(3 * time.Second)

	select {
	case <-ctx.Done():
		fmt.Println("Operation canceled due to context deadline.")
	default:
		fmt.Println("Operation completed within the deadline.")
	}
}

func main() {
	deadline := time.Now().Add(2 * time.Second)
	deadlineCtx, cancel := context.WithDeadline(context.Background(), deadline)
	defer cancel()

	operation(deadlineCtx)
}
// Operation deadline: 2024-08-31 12:21:15.279442 +0800 CST m=+2.000158542
// Operation canceled due to context deadline.
```

### 3.4 值传递
- 通过ctx value传递userId
```go
func processRequest(ctx context.Context, requestID int) {
	userID, ok := ctx.Value("id").(int)
	if !ok {
		fmt.Println("failed to get id from context.")
		return
	}
	fmt.Printf("processing request %d for user %d\n", requestID, userID)
}

func main() {
	parentCtx := context.WithValue(context.Background(), "id", 123)

	var wg sync.WaitGroup
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(requestID int) {
			childCtx := context.WithValue(parentCtx, "requestID", requestID)
			processRequest(childCtx, requestID)
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

## 4. 源码分析
### 4.1 emptyCtx
context包中定义了一个空的context， 名为emptyCtx，用于context的根节点，空的context只是简单的实现了Context，本身不包含任何值，仅用于其他context的父节点。
```go
// 空的ctx本质上一个整型，它不会被取消、没有值，也没有过期时间
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key any) any {
	return nil
}
```

emptyCtx通过下面两个导出的函数（首字母大写）对外公开：我们所常用的 context.Background()  和 context.TODO() 方法。本质上二者并无差别
```go
// context.Background()函数返回一个空对象，被视为所有上下文树的根节点，不需要传递值或取消信号。
func Background() Context {return background}
//context.TODO()函数返回一个空对象，用于该部分代码还未确定具体需要哪种上下文对象，
func TODO() Context {return todo}
```

### 4.2 valueCtx
valueCtx在空ctx的基础上，增加了key-val键值对，用于保存一些数据供上下文使用。在实际使用过程中通过WithValue()函数构造。
- `valueCtx` 结构体定义
```go
// 通过组合Context的方式，携带一个key-val对
type valueCtx struct {
	Context
	key, val any
}
```

- `WithValue()` 源码
```go
func WithValue(parent Context, key, val any) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```
通过以上代码可以看出，valueCtx的每次构建，都是在上一个ctx的基础上生成一个新的ctx，每一个valueCtx只有一个键值对，多个键值对构成一个串型的数据结构。
![](/img/Golang-Context解析/2.png)   

- `valueCtx.Value()`函数
Value()函数从当前的ctx开始找key的值，如果找不到，继续找父ctx，直至找到emptyCtx为止
```go
func (c *valueCtx) Value(key any) any {
	if c.key == key {
		return c.val
	}
	return value(c.Context, key)
}

func value(c Context, key any) any {
	// 从底层循环往父层寻找指定key的值
	for {
		switch ctx := c.(type) {
		case *valueCtx:
			if key == ctx.key {
				return ctx.val
			}
			c = ctx.Context
		// cancelCtxKey 是一个特殊的key，如果属于cancelCtx或者timerCtx且key为特殊key，则返回这个cancelCtx
		case *cancelCtx:
			if key == &cancelCtxKey {
				return c
			}
			c = ctx.Context
		case *timerCtx:
			if key == &cancelCtxKey {
				return ctx.cancelCtx
			}
			c = ctx.Context
		case *emptyCtx:
			return nil
		default:
			return c.Value(key)
		}
	}
}
```
- 特点
    - 一个 valueCtx 只能存一个 kv 对，n 个 kv 对会嵌套 n 个 valueCtx，造成空间浪费，不适合大量存储；
    - 基于 k 寻找 v 的过程是线性的，时间复杂度 O(N)；
    - 不支持基于 k 的去重，相同 k 可能重复存在，并基于起点的不同，返回不同的 v.
    - valueCtx结构对于Context接口只重写了Value方法，其他三个方法直接调用嵌入的Context。
### 4.3 cancelCtx
context包中第三个比较重要的ctx是cancelCtx，顾名思义是可以取消的context，该context在构建时返回一个取消函数，可以供主动取消。

- `cancelCtx`结构定义
```go
type cancelCtx struct {
	Context                        // 嵌入的Context

	mu       sync.Mutex            // 互斥锁，保护以下字段
	done     atomic.Value          // 原子通道，惰性创建,第一次被cancel调用关闭
	children map[canceler]struct{} // 存储子树中第一个可以被cancel的ctx
	err      error                 // 存储error信息
	cause    error                 // 存储带有原因的error信息
}
```

- `cancelCtx` 主要方法
    cancelCtx作为Context接口的实现，重写了Done、Err和Value方法，分别简单介绍如下
    ```go
// Done 方法返回一个通道，使用了惰性加载的机制，只有第一次调用Done方法时才会被创建
// 将通道放到atomic.Value中 保证通道操作的原子性
func (c *cancelCtx) Done() <-chan struct{} {
	d := c.done.Load()
	if d != nil {
		return d.(chan struct{})
	}
	// 加锁的方式做二次检查，避免多并发场景下被重复创建
	c.mu.Lock()
	defer c.mu.Unlock()
	d = c.done.Load()
	if d == nil {
		d = make(chan struct{})
		c.done.Store(d)
	}
	return d.(chan struct{})
}

// Value方法复用valueCtx的递归逻辑，只是有一种特殊的处理情况
// 当key是cancelCtxKey的时候，返回的是cancelCtx本身
func (c *cancelCtx) Value(key any) any {
	if key == &cancelCtxKey {
		return c
	}
	return value(c.Context, key)
}

// Err 返回结构体err
// cancelCtx.err 默认是nil，在被cancel的时候指定一个error变量：“context canceled”
func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}
```

- `WithCancel()` 构造方法
WithCancel是context官方包对外提供的函数之一，函数接受一个父上下文对象parent 作为参数，返回一个新的上下文cancelCtx对象ctx 及其对应的取消函数cancel 。当调用取消函数时，该ctx对象及其所有后代ctx对象均会被取消。核心代码逻辑介绍如下：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := withCancel(parent)
	// 将构造的cancelCtx返回，同时返回终止该cancelCtx的闭包函数cancel；第一个参数是 true，也就是说取消的时候，需要将自己从父节点里删除。
	return c, func() { c.cancel(true, Canceled, nil) }
}

func withCancel(parent Context) *cancelCtx {
	// 如果父ctx为空，则无法构建
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	// 构建一个新的cancelCtx
	c := newCancelCtx(parent)
	// 核心逻辑，构建cancel传播链
	propagateCancel(parent, c)
	return c
}

```
由于cancelCtx的对象在取消时，要同步取消其后代Ctx对象，因此在cancelCtx的构建时，要进行cancel信息的控制链传递，建立父子关系，将本次新建的cancelCtx放在链路中上一个cancelCtx的children map中。如果一个cancelCtx对象的后代不是cancelCtx对象，那这个后代的done channel本身就是cancelCtx的done channel，无需额外关联。因此在控制链的传递中，只需关联cancelCtx的父子关系即可。

```go
// parent为父ctx，child为本次创建的cancelCtx
func propagateCancel(parent Context, child canceler) {
	done := parent.Done()
	// 如果父ctx不会被取消，直接返回即可
	if done == nil {
		return 
	}

	select {
	case <-done:
		// 如果父ctx已被取消则直接中止本次构建的cancelCtx，并用父ctx的取消原因作为子ctx的取消原因
		child.cancel(false, parent.Err(), Cause(parent))
		return
	default:
	}
	
	// 寻找parent ctx的第一个cancelCtx祖先(这个可以是parent自己,由cancelCtx.Value函数的实现决定)，本质上也就是找child ctx的第一个cancelCtx的祖先
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()
		if p.err != nil {
			// 如果父ctx已被取消则直接中止本次构建的cancelCtx
			child.cancel(false, p.err, p.cause)
		} else {
			// 将本次新构建的child ctx加入到最近的cancelCtx祖先的children中
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
		// 如果parent不是cancelCtx，但是又存在cancel的能力，则启动一个协程监控parent的状态，如果parent终止，也及时终止child ctx，并传递parent的error
		goroutines.Add(1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err(), Cause(parent))
			case <-child.Done():
			}
		}()
	}
}

// parentCancelCtx 返回输入parent的第个cancelCtx祖先，可能是parent自己
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	// parent.Done() 返回控制链中第一个实现非空Done()方法的Context
	// 在valueCtx、cancelCtx、timerCtx三者中，只有cancleCtx实现了非空的Done方法，因此parent.Done正常情况下会返回第一个祖先cancelCtx的done channel。但如果Context树中有第三方实现的Context接口实例时，parent.Done可能返回其他channel
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}
	
	// 通过cancelCtxKey找到第一个祖先cancelCtx
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	// 如果p.done不等于done，说明控制链中碰到的第一个实现非空Done()的Context是第三方Context，不是cancelCtx
	pdone, _ := p.done.Load().(chan struct{})
	if pdone != done {
		return nil, false
	}
	return p, true
}
```
![](/img/Golang-Context解析/3.png)   
如上图，由C3生成C4时，C3的parentCancelCtx是C1，C3调用Done()方法返回的也是C1的 done channel，因此将C4放入C1的children中

- `cancelCtx.cancel()` 方法
```go
func (c *cancelCtx) cancel(removeFromParent bool, err, cause error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	if cause == nil {
		cause = err
	}
	c.mu.Lock()
	// 如果已经被取消，直接返回
	if c.err != nil {
		c.mu.Unlock()
		return 
	}
	
	// 记下错误信息，并关闭channel
	c.err = err
	c.cause = cause
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	// 级联取消child cancelCtx
	for child := range c.children {
		child.cancel(false, err, cause)
	}
	c.children = nil
	c.mu.Unlock()

	// 将当前ctx从ctx的父ctx中移除
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```
- `WithCacelCause()` 支持传递取消原因的cancelCtx
早期的cancelCtx在cancel时，能写入的err信息及其有限，只有超时取消和外部取消，Go  1.20版本新增一个可以设置取消原因的方法，他在调用cancel的时候，可以传递一个error参数。
```go
func WithCancelCause(parent Context) (ctx Context, cancel CancelCauseFunc) {
	c := withCancel(parent)
	return c, func(cause error) { c.cancel(true, Canceled, cause) }
}

ctx, cancelFunc := context.WithCancelCause(parentCtx)
defer cancelFunc(errors.New("原因"))
```
### 4.4 timerCtx
timerCtx在cancelCtx的基础上进行了一次封装，除了继承cancelCtx的能力外，新增了一个time.Timer的计时器用于定时终止context，另外新增了一个deadline字段用于timerCtx的过期时间。
- `timerCtx` 结构体
```go
type timerCtx struct {
	*cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time
}

// 重写Context接口的Deadline()方法
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```
- `WithDeadline()` 构造方法
```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	// 校验parent ctx是否为空
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	// 校验parent的过期时间是否比自己早，如果比自己早，直接构造一个基于parent的cancelCtx即可
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}
	// 构造新的timerCtx
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	// ctx控制链的取消关系同步，方式与cancelCtx一样
	propagateCancel(parent, c)
	// 判断过期时间是否已到，若是，直接 cancel timerCtx，并返回 DeadlineExceeded 的错误；
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded, nil) // deadline has already passed
		return c, func() { c.cancel(false, Canceled, nil) }
	}
	// 启动定时器，时间到了后取消该ctx
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded, nil)
		})
	}
	return c, func() { c.cancel(true, Canceled, nil) }
}
```
- `WithTimeout()` 构造方法
```go
// WithTimeout 直接复用WithDeadline方法，在当前时刻加上时间段就是最后的超时时间
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```
- `timerCtx.cancel()` 方法
```go
func (c *timerCtx) cancel(removeFromParent bool, err, cause error) {
	// 直接复用cancelCtx的取消能力
	c.cancelCtx.cancel(false, err, cause)
	// 从parent ctx中移除父子关系
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	// 停止计时器
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```
## 5. 注意事项
- 不要在结构类型中加入 Context 参数，而是将它显式地传递给需要它的每个函数，并且它应该是第一个参数，通常命名为 ctx:
- Context 是线程安全的，可以放心地在多个 goroutine 中使用。
- 当你把 Context 传递给多个 goroutine 使用时，只要执行一次 cancel 操作，所有的 goroutine 就可以收到 取消的信号
- 不要把原本可以由函数参数来传递的变量，交给 Context 的 Value 来传递。
    - ctx一般只存储请求维度的数据，比如用户信息、logId等，只保留告知类型的数据，而不保留控制性质的数据，即不存储业务逻辑上的控制参数数据。
- 当一个函数需要接收一个 Context 时，但是此时你还不知道要传递什么 Context 时，可以先用 context.TODO 来代替，而不要选择传递一个 nil。
- 当一个 Context 被 cancel 时，继承自该 Context 的所有 子 Context 都会被 cancel。

## 6. 参考资料

[Golang context实现原理与源码分析](https://juejin.cn/post/7265880174381727759)

[Golang 笔记](https://blog.51cto.com/u_15533611/5202001)

[Go语言并发编程-上下文Context](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/) 
