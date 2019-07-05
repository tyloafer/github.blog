---
title: 深入理解Go-逃逸分析
tags:
  - Go
categories:
  - Go
date: '2019-07-04 20:04'
abbrlink: 30194
---

> How do I know whether a variable is allocated on the heap or the stack?
>
> From a correctness standpoint, you don't need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
>
> The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
>
> In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

在Go里面定义了一个变量，到底是分配在堆上还是栈上，Go官方文档告诉我们，不需要管，他们会分析，其实这个分析就是逃逸分析

在编程语言的编译优化原理中，分析指针动态范围的方法称之为逃逸分析。通俗来讲，当一个对象的指针被多个方法或线程引用时，我们称这个指针发生了逃逸。

<!--more-->

发生逃逸行为的情况主要有两种：

- 方法逃逸：当一个对象在方法中定义之后，作为参数传递或返回值到其它方法中
- 线程逃逸：如类变量或实例变量，可能被其它线程访问到

这里主要对 **方法逃逸** 进行分析，通过逃逸分析来判断一个变量到底是分配在堆上还是栈上



# 逃逸策略

- 如果编译器不能证明某个变量在函数返回后不再被引用，则分配在堆上
- 如果一个变量过大，则有可能分配在堆上

# 分析目的

- 不逃逸的对象分配在栈上，则变量在用完后就会被编译器回收，从而减少GC的压力
- 栈上的分配要比堆上的分配更加高效
- 同步消除，如果定义的对象上有同步锁，但是栈在运行时只有一个线程访问，逃逸分析后如果在栈上则会将同步锁去除

# 逃逸场景

## 指针逃逸

> 在 build 的时候，通过添加 -gcflags "-m" 编译参数就可以查看编译过程中的逃逸分析

在有些时候，因为变量太大等原因，我们会选择返回变量的指针，而非变量，这里其实就是逃逸的一个经典现象

~~~go
func main() {
	test()
}

func test() *int {
	i := 1
	return &i
}
~~~

逃逸分析结果：

~~~
# command-line-arguments
./main.go:7:6: can inline test
./main.go:3:6: can inline main
./main.go:4:6: inlining call to test
./main.go:4:6: main &i does not escape
./main.go:9:9: &i escapes to heap
./main.go:8:2: moved to heap: i
~~~

可以看到最后两行指出，变量 `i` 逃逸到了 `heap` 上

## 栈空间不足逃逸

首先，我们尝试创建一个 长度较小的 slice

~~~go
func main() {
	stack()
}

func stack() {
	s := make([]int, 10, 10)
	s[0] = 1
}
~~~

逃逸分析结果：

~~~
./main.go:12:6: can inline stack
./main.go:3:6: can inline main
./main.go:4:7: inlining call to stack
./main.go:4:7: main make([]int, 10, 10) does not escape
./main.go:13:11: stack make([]int, 10, 10) does not escape
~~~

结果显示未逃逸



然后，我们创建一个超大的slice

~~~go
func main() {
	stack()
}

func stack() {
	s := make([]int, 100000, 100000)
	s[0] = 1
}
~~~

逃逸分析结果：

~~~
./main.go:12:6: can inline stack
./main.go:3:6: can inline main
./main.go:4:7: inlining call to stack
./main.go:4:7: make([]int, 100000, 100000) escapes to heap
./main.go:13:11: make([]int, 100000, 100000) escapes to heap
~~~

这时候就逃逸到了堆上了

## 动态类型逃逸

~~~go
func main() {
	dynamic()
}

func dynamic() interface{} {
	i := 0
	return i
}
~~~

逃逸分析结果：

~~~
./main.go:18:6: can inline dynamic
./main.go:3:6: can inline main
./main.go:5:9: inlining call to dynamic
./main.go:5:9: main i does not escape
./main.go:20:2: i escapes to heap
~~~

这里的动态类型逃逸，其实在理解了`interface{}`的内部结构后，还是可以归并到 **指针逃逸** 这一类的，有兴趣的同学可以看一下 [《深入理解Go的interface》](https://tyloafer.github.io/posts/2647/)

## 闭包引用逃逸

~~~go
func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		f()
	}
}
func fibonacci() func() int {
	a, b := 0, 1
	return func() int {
		a, b = b, a+b
		return a
	}
}
~~~

逃逸分析结果：

~~~
./main.go:11:9: can inline fibonacci.func1
./main.go:11:9: func literal escapes to heap
./main.go:11:9: func literal escapes to heap
./main.go:12:10: &b escapes to heap
./main.go:10:5: moved to heap: b
./main.go:12:13: &a escapes to heap
./main.go:10:2: moved to heap: a
~~~



# 参考

[《Go 逃逸分析》](https://my.oschina.net/renhc/blog/2222104)

