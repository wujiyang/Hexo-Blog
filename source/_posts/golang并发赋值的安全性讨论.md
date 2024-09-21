---
title: Golang并发赋值的安全性分析
date: 2024-09-21 19:47:51
index_img: /img/golang并发赋值的安全性讨论/1.jpg
tags:
    - 并发安全
    - 并发赋值
categories:
    - Golang
		- 并发编程
---

> 声明：本文非原创，主要参考恋喵大鲤鱼的博客：https://blog.csdn.net/K346K346/article/details/115099353 ，稍微做些修改。

<!-- more -->  

##  1. 什么是并发安全
并发安全就是程序在并发情况下执行的结果是正确的。
比如对一个变量简单的自增操作count++，在非并发下很好理解，而在并发情况下却容易出现预期之外的结果，这样的代码就是非并发安全的。
因为count++其实是分成两步执行的，当分成了两步执行，那么在并发时多协程就可以趁着这个时间间隙作怪。
比如a、b两个协程同时执行执行count++，正常情况下应该返回3，但实际返回了2
```go
count:= 1
a > 读取count : 1
b > 读取count : 1
a > 计算count+1 : 2
b > 计算count+1 : 2
a > 赋值count : 2
b > 赋值count : 2
```

##  2. Struct并发安全吗
💡 结论：  

**多字段结构体并发赋值不安全，但是单字段结构体并发赋值是安全的。**
### 2.1 现象
对简单的一个变量进行自增都会出现并发问题，那对一个结构体赋值会不会出现问题呢。基于此，设计一个两个字段的结构体，高频的进行并发赋值并观察结果。
```go
type Test struct {
	X int
	Y int
}

func main() {
	var g Test

	for i := 0; i < 1000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = Test{1, 2}
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = Test{3, 4}
		}()
		wg.Wait()

		// 赋值异常判断
		if !((g.X == 1 && g.Y == 2) || (g.X == 3 && g.Y == 4)) {
			fmt.Printf("concurrent assignment error, i=%v g=%+v\n", i, g)
			break
		}
	}
}
```
运行此段代码，发现偶尔出现以下报错：
```go
concurrent assignment error, i=199242 g={X:1 Y:4}
```
### 2.2 分析
对含有多个字段的结构体进行并发赋值时，可能会出现数据错乱的问题：协程1赋值了X字段，而协程2赋值了Y字段。
原因：struct进行字段赋值时并不是原子操作，各个字段的赋值操作是独立进行的，在并发的情况下会出现异常。

扩展：如果一个struct只有一个字段，那并发赋值时就是安全的。

##  3. 如何保证并发安全
Golang提供一个原子类型的atomic.Value来保证并发赋值安全
```go
// A Value provides an atomic load and store of a consistently typed value.
// The zero value for a Value returns nil from Load.
// Once Store has been called, a Value must not be copied.
//
// A Value must not be copied after first use.
type Value struct {
	v interface{}
}
```
使用样例：通过Load和Store方法进行数据读取和存储，保证并发赋值安全
```go
func main() {
	var v atomic.Value

	for i := 0; i < 1000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			v.Store(Test{1, 2})
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			v.Store(Test{3, 4})
		}()
		wg.Wait()

		// 赋值异常判断
		g := v.Load().(Test)
		if (g.X == 1 && g.Y == 2) || (g.X == 3 && g.Y == 4) {
		} else {
			fmt.Printf("concurrent assignment error, i=%v g=%+v", i, g)
			break
		}
	}
}
```

##  4. 数据类型分析
### 4.1 基本数据类型
💡 结论：
- 字节型、布尔型、整形、浮点型、字符型：并发安全
- 复数型：不安全
- 字符串：不安全
- 底层结构为struct类型的数据，一般并发赋值时都不安全；但不安全不代表一定会发生错误；
#### 4.1.1 字节型、布尔型、整型、浮点型、字符型
由于字节型、布尔型、整型、浮点型、字符型的位宽不会超过 64 位，在 64 位的指令集架构中可以由一条机器指令完成，不存在被细分为更小的操作单位，所以这些类型的并发赋值是安全的。
以浮点型为例：
```go
func main() {
	var g float64

	for i := 0; i < 1000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = 1.1
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = 2.2
		}()
		wg.Wait()

		// 赋值异常判断
		if g != 1.1 && g != 2.2 {
			fmt.Printf("concurrent assignment error, i=%v g=%+v", i, g)
			break
		}
	}
}
```
#### 4.1.2 复数型
复数型分为实部和虚部两个部分，二者的赋值是分开的，按照上面的分析说明并发赋值非安全的。
```go
func main() {
	var g complex64

	for i := 0; i < 1000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = complex(1, 2)
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = complex(3, 4)
		}()
		wg.Wait()

		// 赋值异常判断
		if g != complex(1, 2) && g != complex(3, 4) {
			fmt.Printf("concurrent assignment error, i=%v g=%+v \n", i, g)
			break
		}
	}
}
```
偶尔会得到如下的效果：
```go
concurrent assignment error, i=466735 g=(1+4i)
```
#### 4.1.3 字符串
在Go中，string本质上是一个只读的字节数组切片，他有几个重要特点：
- string可以为空，长度为0但不是nil，而是""
- string对象不可修改，无法通过索引直接修改字符串内容，需要修改时可以通过[]byte数组转换等方式实现。通常情况下，如果要修改一个字符串，必须创建一个新的字符串。
在源码包src/runtime/string.go我们可以找到 string 的底层数据结构：
```go
type stringStruct struct {
	str unsafe.Pointer // str为字符串首地址 
	len int            // len为字符串的长度
}
```
因为string的底层结构时一个多字段的struct，所以并发赋值是不安全的。
```go
func main() {
	var s string

	for i := 0; i < 1000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			s = "ab"
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			s = "cde"
		}()
		wg.Wait()

		// 赋值异常判断
		if s != "ab" && s != "cde" {
			fmt.Printf("concurrent assignment error, i=%v s=%v\n", i, s)
			break
		}
	}
}
```
偶现如下结果：
```go
concurrent assignment error, i=348275 s=cd
```
📌 注意：

- 底层结构是struct的类型，一般情况下并发赋值都是不安全的
- 并发赋值不安全不代表一定会出错，只是有可能出错；比如上面的string赋值中，如果并发赋两个床度相同的变量，可以等同退化为只有一个字段的结构体，变成并发安全的场景。
### 4.2 复合数据类型
💡 结论：
- 指针：安全
- 函数：安全
- 数组、切片、映射、通道、接口：不安全
- 底层结构为struct类型的数据，一般并发赋值时都不安全；但不安全不代表一定会发生错误；
#### 4.2.1 指针
指针是一个变量，它保存的是另一个变量的内存地址。指针的零值为nil。

因为指针存储的是内存地址，所以位宽为 32位（x86平台）或 64位（x64平台），赋值操作由一个机器指令即可完成，不会被中断，所以也不会出现并发赋值不安全的情况。
这在上面讨论 string 等长不同值并发赋值时，已经验证没有问题。
#### 4.2.2 函数
Go中，函数可以像值一样进行传递，用户也可以定义函数类型。
常规的函数定义如下：
```go
func some_func_name(arguments) return_values
```
定义一个函数类型时，去掉函数名即可
```go
type MyFunc func(arguments) return_values
```
对函数类型进行赋值时，实际赋的是函数地址，一条机器指令即可完成，所以并发赋值是安全的。
```go
type MyFunc func(int, int) int

func main() {
	var g MyFunc

	var i int
	for ; i < 10000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = func(x, y int) int {
				return x + y
			}
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = func(x, y int) int {
				return x - y
			}
		}()
		wg.Wait()

		// 赋值异常判断
		if !(g(1, 1) == 2 || g(1, 1) == 0) {
			fmt.Printf("concurrent assignment error, i=%v g=%+v", i, g)
			break
		}
	}
	if i == 10000000 {
		fmt.Println("no error")
	}
}
```
我们使用unsafe.Sizeof()可以查看函数类型的宽度（字节）。
```go
type Add func(int, int) int
var add Add
fmt.Println(unsafe.Sizeof(add)) 
// 输出 8
```
#### 4.2.3 数组、切片、映射、通道、接口
数组、切片、映射、通道、接口，这些复合类型，除了数组，其他底层数据结构都是 struct，所以并发都不是安全的，当然数组并发赋值也是不安全的。
- 数组
数组是相同类型值的集合，数组的长度是其类型的一部分。数组赋值和传参都会拷贝整个数组的数据，所以数组不是引用类型；
数组的底层数据结构就是其本身，是一个相同类型不同值的顺序排列。所以如果数组位宽不大于 64 位且是 2 的整数次幂（8，16，32，64），那么其并发赋值是安全的，只不过大部分情况并非如此，所以其并发赋值是不安全的。
以字节数组为例，当位宽不超过64时：并发赋值安全
```go
func main() {
	var g [4]byte

	var i int
	for ; i < 10000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = [...]byte{1, 2, 3, 4}
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = [...]byte{3, 4, 5, 6}
		}()
		wg.Wait()

		// 赋值异常判断
		if !(g == [...]byte{1, 2, 3, 4} || g == [...]byte{3, 4, 5, 6}) {
			fmt.Printf("concurrent assignment error, i=%v g=%+v", i, g)
			break
		}
	}
	if i == 10000000 {
		fmt.Println("no error")
	}
}
```
可以看到，位宽为 32 位的数组 [4]byte，虽然有四个元素但是赋值时由一条机器指令完成，所以也是原子操作。

如果把字节数组的长度换成下面这样子，即使没有超过 64 位，也需要多条指令完成赋值，因为 CPU 中并没有这样位宽的寄存器，需要拆分为多条指令来完成。
```go
[3]byte
[5]byte
[7]byte
```
- 切片
slice 也是相同类型值的集合，只不过切片是动态调整大小的，内部是对数组的引用，相当于动态数组。如上所述，数组的大小是固定的，因此切片为数组提供了更灵活的接口。
切片是一种引用类型，它内部由三个字段表示：
- 数组地址
- 数组长度
- 容量大小
在源码包src/runtime/slice.go我们可以找到切片的底层数据结构：
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
因为其是一个 struct，所以并发赋值是不安全的.
- 映射
map是内置的k-v数据结构，是一个同种类型的无序组，内部元素可以通过唯一键进行索引，map的底层也是一个结构体，定义位于src/runtime/map.go。对map的并发读写会引起panic
```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```
- 通道
channel 在 goroutine 之间提供同步和通信。可以将其视为 goroutines 通过其发送值和接收值的管道。操作符<-用于发送或接收数据，箭头方向指定数据流的方向。
```go
ch <- val    	// Sending a value present in var variable to channel
val := <-cha	// Receive a value from  the channel and assign it to val variable
```
channel作为数据同步的通道，他在数据读取和发送的过程中，channel 是并发安全的。多个 goroutine 可以安全地从同一个 channel 发送或接收数据，而不需要使用额外的同步机制（如锁）。

本文讨论的是对象赋值的安全性，即将一个channel赋值给另外一个channel，这个操作一般不会出现，但是如果一定要进行分析的话，这种操作是并发非安全的，因为它的底层也是一个struct。
```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
- 接口
接口是Go中的一个类型，一般它是一组方法的集合。任意实现该接口所有方法的类型都属于该接口类型。接口的零值为nil。
定义一个接口类型的变量后，如果有具体类型A实现了该接口的方法，则可以将任何A类型的值赋值给这个变量。
Go中有一个特殊的情况，就是空接口，它不包含任何方法。所以，默认情况下，任何具体类型都实现空接口。
1. 空接口的底层结构为runtime.eface
2. 非空接口的底层结构为runtime.iface
```go
type eface struct { // 16 字节
	_type *_type
	data  unsafe.Pointer
}

type iface struct { // 16 字节
	tab  *itab
	data unsafe.Pointer
}
```
由接口底层结构可以看到：
- 接口底层数据结构包含两个字段，相互赋值时如果是相同具体类型不同值并发赋给一个接口，那么只有一个字段 data 的值是不同的，此时退化成指针的并发赋值，所以是安全的。但如果是不同具体类型的值并发赋给一个接口，那么并引发 panic。
- 不同类型验证代码如下：
```go
func main() {
	var g interface{}

	var i int
	for ; i < 10000000; i++ {
		var wg sync.WaitGroup
		// 协程 1
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = "a"
		}()

		// 协程 2
		wg.Add(1)
		go func() {
			defer wg.Done()
			g = 2
		}()
		wg.Wait()

		// 赋值异常判断
		v1, _ := g.(string)
		v2, _ := g.(int)
		if v1 != "a" && v2 != 2 {
			fmt.Printf("concurrent assignment error, i=%v g=%v ", i, g)
			break
		}
	}
	if i == 10000000 {
		fmt.Println("no error")
	}
}
```
执行时会报错：
```go
anic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x2 addr=0x2 pc=0x10297598c]
```
更多关于interface的分析，参考：《Golang Interface解析》
## 5. 小结
Go 多协程并发的场景无处不在，并发对同一变量的赋值也是经常遇到。本文尝试探讨了 Go 中所有类型并发赋值的安全性。
- 由一条机器指令完成赋值的类型并发赋值是安全的，这些类型有：字节型，布尔型、整型、浮点型、字符型、指针、函数。
- 数组由一个或多个元素组成，大部分情况并发不安全。注意：当位宽不大于 64 位且是 2 的整数次幂（8，16，32，64），那么其并发赋值是安全的。
- struct 或底层是 struct 的类型并发赋值大部分情况并发不安全，这些类型有：复数、字符串、 数组、切片、映射、通道、接口。注意：当 struct 赋值时退化为单个字段由一个机器指令完成赋值时，并发赋值又是安全的。这种情况有：
    - 实部或虚部相同的复数的并发赋值；
    - 等长字符串的并发赋值；
    - 同长度同容量切片的并发赋值；
    - 同一种具体类型不同值并发赋给接口。

原文链接：https://blog.csdn.net/K346K346/article/details/115099353