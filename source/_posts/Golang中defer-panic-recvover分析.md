---
title: Golang中的Defer、Panic和Recover
date: 2024-07-13 16:04:50
index_img: /img/Golang中defer-panic-recvover分析/0.jpg
tags:
    - Golang
    - 异常处理
categories:
    - Golang
---

Golang中没有类似java的try catch机制进行异常处理，而是引入了defer、panic和recover来触发异常和终止异常。

<!-- more -->  

## 1. Defer
defer是go语言提供的一种用于注册延迟调用的机制：让函数或者语句可以在当前函数执行完毕后执行（包括通过return正常结束或者panic导致的异常结束）。

常用于一些操作后的收尾工作，比如关闭连接、释放锁、关闭文件等，使用起来比较优雅。  
``` golang 
f, _ := os.Open("defer.txt")
defer f.Close()  
```

### 1.1 Defer基本原理  
defer使用准则
- 每次defer语句执行的时候，会把函数“压栈”，函数参数会被拷贝下来，但是闭包引用不会
    - 作为函数参数：在defer定义时就把值传递给defer，并被cache起来，后续不会再改变
    - 作为闭包引用：在defer函数真正调用时根据整个上下文确定当前的值。
- 当外层逻辑退出时，defer函数按照定义的逆序执行；【先入后出，压栈进行】
- 恰当的使用defer方法可以修改返回值  

- 先入后出
```golang
func a() {
	for i := 1; i <= 3; i++ {
		defer fmt.Println(i)
	}
} 
// 输出 3 2 1
```
- defer函数的参数引用方式
    - 作为函数入参，在函数定义时就已经传递给defer函数，后续不会再改变
    ```go
    func b() {
        for i := 1; i <= 3; i++ {
            defer func(a int) {
                fmt.Println(a)
            }(i)
        }
    }
    // 输出 3 2 1
    ```
    - 作为闭包的引用时，在最后执行的时候确定上下文的值
    ```go
    func c() {
        for i := 1; i <= 3; i++ {
            defer func() {
                fmt.Println(i)
            }()
        }
    }
    // 输出 4 4 4 
    // i = 3 时定义最后一个defer，又进行了+1操作后，才真正开始执行defer函数
    ```

### 1.2 defer命令拆解

上面说defer可以操作返回值，本质上就是在含有defer函数的代码中，一个 return xxx的语句可以进行拆解，这条语句在实际执行时分三步进行：
> 1. 返回值 = xxx
> 2. 执行defer函数
> 3. 空的return  

典型案例如下：
```go
func d() (r int) {
	t := 5
	defer func() {
		t = t + 5
	}()
	return t
}
// 返回 5
// r=t; t=t+5; return 

func e() (r int) {
	defer func(r int) {
		r = r + 5
	}(r)
	return 5
}
// 返回 5
// r = 5； r=r+5（这个r为形参r，不是返回值r）； return

func f() (r int) {
	defer func() {
		r = r + 5
	}()
	return 5
}

// 返回 10
// r = 5； r=r+5； return
```

从以上的实际案例中可以看到：

📌 <font color="red">能通过defer修改返回值的场景，一定是有命名返回值的函数场景。</font>  

## 2. Painc和Recover
当go在运行过程中发送异常时，Go运行时会触发运行时panic，并在调用它的函数中向本层以及所有上层逐级抛出，若一直没有recover捕获，程序最终会终止。  

若在某层defer语句中被recover捕获，控制流程将进入到recover之后的语句中。通过panic、defer和recover，可以实现类似try catch的功能。  

例如： 
```go
func g() {
	defer func() {
		fmt.Println("defer in")
		if err := recover(); err != nil {
			fmt.Println(err)
		}
	}()

	fmt.Println("panic begin")
	panic("hello, I'm panic")
	fmt.Println("panic end")
}

// 输出
// panic begin
// defer in
// hello, I'm panic
```
📌 <font color="red">注意：defer中的recover仅对当前协程生效，且仅在直接被defer函数调用才有效</font>    


### 2.1 Recover使用规则 
Recover在使用的过程中，一般会判断返回值是否为空，当返回值为空的时候代表没有正常捕获问题。golang官方介绍了几种Recover返回为空的场景。
1. panic的参数为空
2. 当前协程没有panic
3. recover没有直接被defer函数调用   

分别介绍如下：
- panic参数为空，可以正常捕获 // 本身panic参数为nil时候等同于没有发送panic
```go
func main() {
	defer func() {
		if err := recover(); err != nil {
			log.Printf("panic: %v", err)
		}
	}()

	panic(nil)
}
```

- 跨协程调用recover函数，无法捕获异常
```go
// panic跟recover不在一个协程，无法捕获异常
func main() {
	defer func() {
		log.Println(recover())
	}()

	go func() {
		panic("A bad boy stole a server")
	}()
}
```

- 不在defer函数中直接调用，无法捕获异常
```go
func Test() {
	defer func() {
		// 对 recover进行了一层封装，无法正常生效
		Recover("Panic in goroutine")
	}()
	panic("A bad boy stole the server")
}

func Recover(funcName string) {
	if err := recover(); err != nil {
		log.Printf("panic para: %v, panic info: %v\n", funcName, err)
	}
}
```

- 进阶分析
```go
func main() {
	// recover是在匿名函数/闭包中使用，实际执行时取值，触发panic后有值
	go func() {
		defer func() {
			log.Print(recover(), func() string {
				return ", a handsome police catch him.\n\n"
			}())
		}()
		panic("A bad boy stole a server")
	}()

	time.Sleep(3 * time.Second)
	
	// recover作为函数参数使用，定义时就完成值拷贝，刚开始没报错，最后也不会有值。
	go func() {
		defer log.Print(recover(), func() string {
			return ", but a handsome police catch nothing.\n"
		}())
		panic("A bad boy stole a server, again.")
	}()
	time.Sleep(3 * time.Second)
}
```

### 2.2 Defer、Panic、Recover原理浅析  

![](/img/Golang中defer-panic-recvover分析/2.png)   
异常恢复流程可以总结成以下主要流程：
- 触发panic流程：panic终止程序的过程是由编译器将关键字 panic 转换成 runtime.gopanic() 内置函数。异常恢复的流程基本都在这个函数中：
    - 首先会创建一个_panic结构体用来记录当前panic，并且将当前panic加入当前goroutine的_panic链表
    - 然后循环从当前 goroutine 的_defer链表中获取runtime._defer并调用runtime.reflectcall()运行defer函数。
- 恢复流程：如果defer函数中如果recover调用，recover会被汇编转换成runtime.gorecover调用，该函数会标记该panic已经被recover。在执行完某个defer后，如果该panic被标记为recover，则会调用runtime.recovery恢复goroutine的执行。
- 崩溃流程：如果_defer链表为空，或者执行完所有的defer都不包含recover调用，则会调用runtime.fatalpanic打印panic信息，然后中止/退出程序。

## 3. 参考资料

[https://zhuanlan.zhihu.com/p/487749806](https://zhuanlan.zhihu.com/p/487749806) 

[https://zhuanlan.zhihu.com/p/689615742](https://zhuanlan.zhihu.com/p/689615742)

[https://go101.org/article/panic-and-recover-use-cases.html](https://go101.org/article/panic-and-recover-use-cases.html) 

[https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-panic-recover/)