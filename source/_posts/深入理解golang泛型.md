---
title: 深入理解Golang泛型
date: 2024-03-03 23:43:19
index_img: /img/深入理解golang泛型/0.jpg
tags:
    - Golang
    - 泛型
categories:
    - Golang
---

泛型（Generics）是一种编程思想，它允许在编写代码时使用未知的类型。泛型可以增加代码的灵活性和可复用性，同时还能提高代码的安全性和可读性。Go从1.18开始支持泛型的实现并在后续的版本中逐渐完善。

<!-- more -->  

## 1. Golang泛型
- 举个栗子, 现在有一个a+b的函数  
``` golang  
func add(a int, b int) int{
    return a + b
}
``` 
函数比较简单，一看就是两个int类型的数据相加。现在思考一个问题，在golang泛型出现之前，如果想实现两个float相加该如何处理？一般存在两种方式：
- 再写一个函数   
``` golang  
func add(a float64, b float64) float64{
    return a + b
} 
``` 
- 把函数改造成反射函数  
``` golang   
func Add(a interface{}, b interface{}) interface{} {
    switch a.(type) {
    case int:
        return a.(int) + b.(int)
    case float32:
        return a.(float32) + b.(float32)
    default:
        return nil
    }
}

```

现在golang支持泛型以后，该怎么实现呢？   
``` golang 
func Add[T int | float32 | float64](a, b T) T {
	return a + b
}

func main() {
	fmt.Println(Add(1, 2))
	fmt.Println(Add(1.1, 2.1))
}
``` 
以上就是golang中最新支持的泛型函数。除了泛型函数外，还支持泛型类型和泛型接口。  
- 泛型函数  
- 泛型类型  
- 泛型接口 
## 2. 泛型基本特性
泛型作为和常规类型不同的区别就在于它可以支持多种类型的参数，所以在定义和使用时也有着两个基本的特性：类型参数和类型约束。  
![](/img/深入理解golang泛型/1.jpg)   

### 2.1 类型参数    
- 类型参数是泛型函数或类型的一个占位符，表示一个未知的类型。  
- 类型参数列表出现在常规参数之前。为了区分类型参数列表和常规参数列表，类型参数列表使用方括号[]而不是圆括号()
### 2.2 类型约束  
在使用泛型时，有时需要对泛型类型进行一定的约束。例如，我们希望某个泛型函数或类型只能接受特定类型的参数，或者特定类型的参数必须实现某个接口。在 Go 中，可以使用泛型约束来实现这些需求。   

如上图所示，T就是该泛型函数的参数，any就是参数的类型，达标任意类型即可。   


对于泛型类型的约束方法，常见的方式有如下几种：  
#### 2.2.1 常规类型约束  
``` golang 
func Add[T int | float32 | float64](a, b T) T {
	return a + b
}
``` 
当前这种类型约束方式简单明了，代表函数可以支持int，float32和float64三种参数的类型。 
#### 2.2.2 类型集约束    
类型集表示一堆类型的集合，用来在泛型函数的声明中约束类型参数的范围. 如果需要支持的类型比较多，可以写在类型集中。
``` golang 
// 一堆类型的集合，用来在范型函数的声明中约束类型参数的范围
type numbers interface {
	int | int32 | uint32 | int64 | uint64 | float32 | float64
}

// 约束T可以为number中的任一个元素
func Sub[T numbers](a, b T) T {
	return a - b
}
```
#### 2.2.3 联合约束元素  
联合元素，写成一系列由竖线 ( |) 分隔的约束元素。例如：int | float32或~int8 | ~int16 | ~int32 | ~int64  
-  ~int 代表所有int的衍生类型，比如自定义的别名；type myInt int; myInt就是 ~int类型。  

``` golang  
type Integer interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}

func AddInteger[T Integer](a, b T) T {
	return a + b
}
```
#### 2.2.4 推荐写法
在使用类型推荐时，尽量将类型限制在最简单的基本类型中，比如同样是类型T的数组。 
```golang 
// 推荐
type Student1[T int | string] struct {
	Name string
	Data []T
}

type Student2[T []int | []string] struct {
	Name string
	Data T
}
```

## 3. 泛型实践   
Golang 泛型可以应用于各种数据结构和算法，例如排序、搜索、映射等；也可使用泛型类型实现一些通用的数据结构。
### 3.1 工具函数  
- 排序  
在 Golang 中，使用 sort 包可以对任意类型的切片进行排序。  
```golang 
func Sort[T int | int32 | float32](s []T) {
	sort.Slice(s, func(i, j int) bool {
		return s[i] < s[j]
	})
}
```
- 搜索  
在 Golang 中，使用 search 包可以对任意类型的切片进行搜索.   
```golang 
func Search[T int | int32 | int64](s []T, x T) int {
	return sort.Search(len(s), func(i int) bool {
		return s[i] >= x
	})
}
```
- 映射  
在 Golang 中，使用 map 类型可以实现映射。为了支持泛型映射，我们可以定义一个泛型函数 Map[K comparable, V any]，如下所示：  
```golang 
func Map[K comparable, V any](s []K, f func(K) V) map[K]V {
	result := make(map[K]V)
	for _, k := range s {
		result[k] = f(k)
	}
	return result
}  

words := []string{"apple", "banana", "cherry", "durian", "elderberry", "fig"}
uppercased := generics.Map(words, func(word string) string {
    return strings.ToUpper(word)
})
fmt.Println(uppercased)
```
### 3.2 数据结构
简单的实现一个基于泛型的队列。  
```golang 
// Queue - 队列
type Queue[T any] struct {
	items []T
}

// Put 将数据放入队列尾部
func (q *Queue[T]) Put(value T) {
	q.items = append(q.items, value)
}

// Pop 从队列头部取出并从头部删除对应数据
func (q *Queue[T]) Pop() (T, bool) {
	var value T
	if len(q.items) == 0 {
		return value, true
	}

	value = q.items[0]
	q.items = q.items[1:]
	return value, len(q.items) == 0
}

// Size 队列大小
func (q Queue[T]) Size() int {
	return len(q.items)
}
```

## 4. 接口   
- 在泛型出来之前，接口的定义是：接口是一个方法集   
```golang
type error interface{
	Error() string
}
```
- 在泛型出来之后，接口的定义是：接口是一个类型集  
```golang
type number interface{
	int | float32
}
```
### 4.1 并集、交集、空集
- 并集：使用|连接的就是并集      
```golang
// number 是下列基础类型的并集
type number interface{
	int | int32 | uint32 | int64 | uint64
}
```
- 交集：如果一个接口的定义包含多行类型，就取他们的交集。  
```golang
type Int interface {
    int | int8 | int16 | int32 | int64 | uint | uint8 | uint16 | uint32 | uint64
}

type Uint interface {
    uint | uint8 | uint16 | uint32 | uint64
}

// 接口Status代表 Int和Uint的交集
type Status interface {  
    Int
    Uint
}
```
- 空集：如果多行类型没有交集，就是空集.    
```golang 
// 类型 ~int 和 ~float 没有相交的类型，所以接口 Bad 代表的类型集为空
type Bad interface {
    ~int
    ~float 
} 
``` 
### 4.2 接口类型
- 基本接口: 接口的定义中只有方法    
```golang 
// 接口中只有方法，所以是基本接口
type MyError interface { 
    Error() string
}
```
- 一般接口: 接口的定义中不仅有方法，还有类型    
``` golang
// 接口 Uint 中有类型，所以是一般接口
type Uint interface { 
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

// ReadWriter 接口既有方法也有类型，所以是一般接口
type ReadWriter interface {  
    ~string | ~[]rune

    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}

// 一般接口类型不能用来定义变量，只能用于泛型的类型约束中  

// 错误。Uint是一般接口，只能用于类型约束，不得用于变量定义
var uintInf Uint   
```
- 如何实现一般接口？ 
```golang
// 先定义一个具体的类型，然后再实现具体的方法 

// StringReadWriter 实现了接口 ReadWriter
type StringReadWriter string

func (s StringReadWriter) Read(p []byte) (n int, err error) {
    // ...
}

func (s StringReadWriter) Write(p []byte) (n int, err error) {
    // ...
}
``` 

## 5. 参考资料  

- https://www.kunkkawu.com/archives/shen-ru-li-jie-golang-de-fan-xing  
- https://juejin.cn/post/7229462763947917367 