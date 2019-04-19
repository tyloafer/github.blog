---
title: Go值拷贝的一些思考
tags:
  - Go
categories:
  - Go
date: '2019-4-18 19:45'
abbrlink: 50287
---

> [In a function call, the function value and arguments are evaluated in the usual order. After they are evaluated, the parameters of the call are passed by value to the function and the called function begins execution.](文档地址：https://golang.org/ref/spec#Calls)

官方文档已经明确说明：Go里边函数传参只有值传递一种方式: 值传递
那么为什么会引发Go的值拷贝的讨论？
<!-- more -->
## 现象
用下面的代码做一下展示
~~~
package main

import (
	"fmt"
)

func main() {
	arr := [5]int{0, 1, 2, 3, 4}
	s := arr[1:]
	changeSlice(s)
	fmt.Println(s)
	fmt.Println(arr)
}

func changeSlice(arr []int) {
	for i := range arr {
		arr[i] = 10
	}
}
~~~
输出结果

~~~

[10 10 10 10]
[0 10 10 10 10]
~~~
如果Go是值拷贝的，那么我修改了函数 `changeSlice` 里面的`slice s` 的值，为什么main函数里面的`slice`和 `array`也被修改了

## 原因
![](https://github-1253518569.cos.ap-shanghai.myqcloud.com/address.jpg)

以上图为例，a 是初始变量，b 是引用变量(Go中并不存在)，p 是指针变量
变量a被拷贝后，地址发生了变化，地址上存储的是原先地址存储的值 10
变量p被拷贝后，地址发生了变化，地址上存储的还是原先地址存储的值 ）0X001, 然后按照这个地址去查找，找到的是 0X001 上面存储的值

所以，当你去修改拷贝后的*p的值，其实修改的还是0X001地址上的值，而不是 拷贝后a的值

![](https://github-1253518569.cos.ap-shanghai.myqcloud.com/slice-array.jpg)

那么我们接下来看slice，slice在实现的时候，其实是对array的映射，也就是说slice存对应的是原array的地址，就类似于p与a的关系，那么整个slice拷贝后，拷贝后的slice中存储的还是array的地址，去修改拷贝后的slice，其实跟修改slice，和原array是一样的

## 试验
我们用下面一个例子，实现以下我们上面的想法
~~~
package main

import (
	"fmt"
)

func main() {
	var a *int
	b := 10
	a = &b
	change(a)
	fmt.Println(a, b)
}

func change(a *int) {
	*a = 30
}
~~~
打印结果
~~~
0xc000096000 30
~~~
符合猜想

## More
这里的东西，其实用dlv调试会看的很方便，有兴趣可以动一下手

## 结论
Go的拷贝都是值拷贝，只是slice中存储的是原array的地址，所以在拷贝的时候，其实是把地址拷贝的新的slice，那么此时修改slice的时候，还是根据slice中存储的地址，找到要修改的内容