---
title: Golang Json Unmarshal 大数精度丢失问题
date: 2024-05-08 22:25:35
index_img: /img/Golang-Json-Unmarshal/0.png
tags:
    - Golang
    - JSON
categories:
    - Golang
---

在Golang中，使用Unmarshal将一个Json数字解析为```interface{}```时, int64等大类型数字会存在精度丢失的问题。    

一般来说，长度超过16位的数字易发生数据截断，而且数据值变成了科学计数法。

<!-- more -->   

## 1. 案例介绍

假设有一个长度为17的json大数字
```golang
var str = `{"id":10000649949036001}`
``` 

如果直接使用json的interface进行解析, 会得到 10000649949036000, 而不是我们想要的10000649949036001 

```golang
func ParseWithInterface() {
	var test map[string]interface{}

	if err := json.Unmarshal([]byte(str), &test); err != nil {
		fmt.Println(err)
	}

	fmt.Println(test)

	dataBytes, err := json.Marshal(test)
	if err != nil {
		panic(err)
	}
	fmt.Printf("%+s\n", string(dataBytes))
}

// 输出
// map[id:1.0000649949036e+16]
// {"id":10000649949036000}  

```
## 2. 原因分析  

原因就出在了interface{}上，在golang中，当数据结构未知，使用map[string]interface{}接收反序列化结果时，会使用float64来进行结果解析，而float类型在处理的过程中时会有精度丢失的，一般超过16位就会发生，同时float64在处理的数据的时候，一般超过6位就会变成科学记数法。使用float64对json number进行解析可以在json的官方代码中找到。

>
> // To unmarshal JSON into an interface value,
> // Unmarshal stores one of these in the interface value:
> //
> //	bool, for JSON booleans
> //	float64, for JSON numbers
> //	string, for JSON strings
> //	[]interface{}, for JSON arrays
> //	map[string]interface{}, for JSON objects
> //	nil for JSON null
>

## 3. 解决方案

既然上文已经说了int64反序列化精度丢失的原因是使用了float64进行解析，那么我们只需要在解析时强行指定不使用float64解析即可。

### 3.1 使用Int64解析
在知道具体类型的情况下，可以直接定义结构体为map[string]int64, 这样在解析时，就会使用int64进行解析。

```golang
func ParseWithInt64() {
	var data map[string]int64
	str := `{"id":10000649949036001}`
	err := json.Unmarshal([]byte(str), &data)
	if err != nil {
		fmt.Println(err)
	}

	fmt.Println(data)
}
// 输出
// map[id:10000649949036001]
```

### 3.2 利用UseNumber 
golang json提供了一种额外的解析方式，可以在解析的过程使用指定使用Number类型而不是float64进行解析，这样就可以避免精度丢失的问题。

```golang
// convertNumber converts the number literal s to a float64 or a Number
// depending on the setting of d.useNumber.
func (d *decodeState) convertNumber(s string) (any, error) {
	if d.useNumber {
		return Number(s), nil
	}
	f, err := strconv.ParseFloat(s, 64)
	if err != nil {
		return nil, &UnmarshalTypeError{Value: "number " + s, Type: reflect.TypeOf(0.0), Offset: int64(d.off)}
	}
	return f, nil
}
```

```golang
func ParseUseNumber() {
	var test map[string]interface{}

	d := json.NewDecoder(bytes.NewReader([]byte(str)))
	d.UseNumber()
	if err := d.Decode(&test); err != nil {
		fmt.Println(err)
	}

	fmt.Println(test)
}

// 输出
// map[id:10000649949036001]
```