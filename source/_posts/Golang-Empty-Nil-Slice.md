---
title: Golang Empty 切片 vs Nil 切片
date: 2024-04-10 21:17:53
index_img: /img/Golang-Empty-Nil-Slice/0.png
tags:
    - Golang
    - Slice
categories:
    - Golang
---

切片（Slice），是Golang开发中常用到的一种基础数据结构，和数组比较相似。在日常开发中，经常见到nil切片、空切片和零切片的概念。今天主要分享一下他们之间的区别。

<!-- more -->  

## 1. 基本对比

### 1.1 结论     

empty slice vs nil slice     

- 相同点   
    - they both have zero length and capacity,   
    - they can be used with the same effect in for loops and append functions,
    - and they even look the same when printed.
- 不同点   
    - nil切片和nil比较的结果是true  
    - empty切片和nil比较的结果是false   


### 1.2 nil 切片  
> nil slice: 长度和容量都为0，并且和nil比较的结果true的切片
> 一般可以通过直接定义变量或者new创建   

```golang
var slice1 []int
var slice2 = *new([]int)
fmt.Println(slice1, len(slice1), cap(slice1), slice1 == nil)
fmt.Println(slice2, len(slice2), cap(slice2), slice2 == nil)  
// [] 0 0 true
// [] 0 0 true
```

可以简单理解为nil切片是只声明但是未初始化的切片。   
  
### 1.3 empty 切片  
> empty slice: 长度和容量都为0，但是和nil比较的结果为false的切片  
> 一般可以通过声明一个长度为0的切片创建  

```golang 
var slice3 = []int{}
var slice4 = make([]int, 0)
fmt.Println(slice3, len(slice3), cap(slice3), slice3 == nil)
fmt.Println(slice4, len(slice4), cap(slice4), slice4 == nil)  
// [] 0 0 false
// [] 0 0 false
```

可以简单理解empty切片是声明后切初始化了长度为0的切片。

### 1.4 零 切片  
> 零切片： 叫法不常见，主要是值长度部位0，但是内部数组元素都是零值或者nil的切片 
> 使用make创建的长度不为0的初始切片就是零切片  

```golang
slice5 := make([]int, 5)  // 0 0 0 0 0
slice6 := make([]*int, 5) // nil nil nil nil nil
fmt.Println(slice5, len(slice5), cap(slice5), slice5 == nil)
fmt.Println(slice6, len(slice6), cap(slice6), slice6 == nil)
// [0 0 0 0 0] 5 5 false
// [nil nil nil nil nil] 5 5 false
```

## 2. 底层原理 
golang的slice底层结构体由3个字段构成：

```golang
type slice struct{
    ptr unsafe.Pointer 
    len int 
    cap int
}

``` 

其中len和cap分别是切片的长度和容量，而ptr则是指向底层数组的指针。    

所谓nil切片，就是ptr指向的指针也为nil  
![](/img/Golang-Empty-Nil-Slice/1.jpg)     



可以通过一个小实验验证指针的数据：  

```golang
var s1 []int  
s2 := []int{}  
s3 := make([]int, 0)

fmt.Printf("s1 (addr: %p): %+8v\n", &s1, *(*reflect.SliceHeader)(unsafe.Pointer(&s1)))
fmt.Printf("s2 (addr: %p): %+8v\n", &s2, *(*reflect.SliceHeader)(unsafe.Pointer(&s2))) 
fmt.Printf("s3 (addr: %p): %+8v\n", &s3, *(*reflect.SliceHeader)(unsafe.Pointer(&s3))) 
// s1 (addr: 0xc0000080d8): {Data:       0 Len:       0 Cap:       0}
// s2 (addr: 0xc0000080f0): {Data:12760120 Len:       0 Cap:       0}
// s3 (addr: 0xc000008108): {Data:12760120 Len:       0 Cap:       0}

```



