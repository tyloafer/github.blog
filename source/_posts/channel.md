---
title: Go学习之Channel总结
tags:
  - Go
categories:
  - Go
date: '2019-06-19 18:04'
abbrlink: 30283
---

Channel是Go中的一个核心类型，你可以把它看成一个管道，通过它并发核心单元就可以发送或者接收数据进行通讯(communication)。

<!--more-->

# 类型

T表示任意的一种类型

- 双向: chan T
- 单向仅发送： chan <-
- 单向仅接受： <- chan

单向的channel，不仅可以通过声明`make(chan <- interface{})` 来创建，还可以通过隐身或显示的通过 `chan` 来转换，如下

~~~go
func main() {
	  channel := make(chan int, 10)
		convert(channel)
}
func convert(channel chan<- int) {}
~~~

在 `convert函数中，就可以吧channel当成单向输入管道来使用了`

既然 双向 chan，既可以接收，也可以发送，为什么还会有单向chan的存在？ 我的一个理解便是 **权限收敛**，例如一个爬虫系统中，有些进程a仅仅负责抓取页面内容，并转发给进程b，那进程a仅需要 `单向发送的chan` 即可

# Blocking

缺省情况下，发送chan或接收chan会一直阻塞着，直到另一方准备好。这种方式可以用来在gororutine中进行同步，而不必使用显示的锁或者条件变量。

如官方的例子中`x, y := <-c, <-c`这句会一直等待计算结果发送到channel中。以下面例子看一下

~~~go
func bufferChannel() {
	channel := make(chan int)
	i := 0
	go func(i int) {
    fmt.Printf("start goroutine %d\n", i)①
		channel <- i
		fmt.Printf("send %d to channel\n", i)②
	}(i)
	time.Sleep(2 * time.Second)
	fmt.Println("sleep 2 second")
	value := <-channel③
	fmt.Println("got ", value)
}
~~~

输出结果如下

~~~
start goroutine 0
sleep 2 second
got  0
send 0 to channel
~~~

可以看出，`go func` 执行到了①后并没有继续执行②，而是等待③执行完成后，再去执行②，也就可以说明 `channel <- i` 阻塞了`goroutine`的继续执行



如果，我不想在这里阻塞，而是我直接把数据放到`channel`里，等接收方准备好后，到`channel`中自取自用如何处理，这里就涉及到了另一个概念 **buffered channel**

# buffered channel

我们把程序修改一下

~~~go
func bufferChannel() {
	channel := make(chan int, 1) // 这里加了个参数
	i := 0
	go func(i int) {
		fmt.Printf("start goroutine %d\n", i)①
		channel <- i
		fmt.Printf("send %d to channel\n", i)②
	}(i)
	time.Sleep(2 * time.Second)
	fmt.Println("sleep 2 second")
	value := <-channel③
	fmt.Println("got ", value)
}
~~~

输出结果

~~~
start goroutine 0
send 0 to channel
sleep 2 second
got  0
~~~

我们发现`go func`执行完①之后就执行了②，并没有等待③的执行结束，这就是**buffered channel**的效果了

我们只需要在make的时候，声明底2个参数，也就是chan的缓冲区大小即可



通过上面的程序可以看出，我们一直在使用③的形成，即`<- chan`来读取chan中的数据，但是如果有多个goroutine在同时像一个chan写数据，我们除了使用

~~~go
for {
	value <- chan
}
~~~

还有什么更优雅的方式吗



# for … range

还是上面那个程序，我们使用 *for … range* 进行一下改造

```go
func bufferChannel() {
   channel := make(chan int, 1)
   i := 0
   go func(i int) {
      fmt.Printf("start goroutine %d\n", i)
      channel <- i
      fmt.Printf("send %d to channel\n", i)
   }(i)
   time.Sleep(2 * time.Second)
   fmt.Println("sleep 2 second")
   for value := range channel {
      fmt.Println("got ", value)
   }
}
```

 这样就可以遍历 `channel` 中的数据了，但是我们在运行的时候就会发现，哎 这个程序怎么停不下来了？`range channel`产生的迭代值为Channel中发送的值，它会一直迭代直到channel被关闭，所以 我们`goroutine`发送完数据后，把`channel`关闭一下试试，这一次，我们不再进行`time.Sleep(2 * time.Second)`

~~~go
func bufferChannel() {
   channel := make(chan int, 1)
   i := 0
   go func(i int) {
      fmt.Printf("start goroutine %d\n", i)
      channel <- i
      fmt.Printf("send %d to channel\n", i)
      close(channel)
   }(i)
   for value := range channel {
      fmt.Println("got ", value)
   }
}
~~~

这样，整个程序就可以正常退出了，所以，在使用`range`的时候需要注意，如果`channel`不关闭，则`range`会一直阻塞在这里的

# select

我们上面讲的一直都是只有一个`channel`的时候，我们应该怎么去做，加入有两个`channel`或者更多的`channel`，我们应该怎么去做，这里就介绍一下 go里面的多路复用 `select`，以下面程序为例

~~~go
func example() {
	tick := time.Tick(time.Second)
	after := time.After(3 * time.Second)
	for {
		select {
		case <-tick:
			fmt.Println("tick 1 second")
		case <-after:
			fmt.Println("after 3 second")
			return
    default:
			fmt.Println("come into default")
			time.Sleep(500 * time.Millisecond)
		}
	}
}
~~~

`time.Tick`是go的time包提供的一个定时器的一个函数，它返回一个`channel`，并在指定时间间隔内，向channel发送一条数据,`time.Tick(time.Second)`就是每秒钟向这个channel发送一个数据

`time.After`是go的time包提供的一个定时器的一个函数，它返回一个`channel`，并在指定时间间隔后，向channel发送一条数据，`time.After(3 * time.Second)`就是3s后向这个channel发送一个数据



输出结果

~~~
come into default
come into default
tick 1 second
come into default
come into default
tick 1 second
come into default
come into default
tick 1 second
after 3 second
~~~

可以看到，`select`会选择一个没有阻塞的 `channel`，并执行响应 `case`下的逻辑，这样就可以避免由于一个 `channel`阻塞而导致后续的逻辑阻塞的情况了



我们继续做个小实验，把上面关闭的`channel`放到 `select`里面试一下

~~~go
func example() {
	tick := time.Tick(time.Second)
	after := time.After(3 * time.Second)
	channel := make(chan int, 1)
	go func() {
		channel <- 1
		close(channel)
	}()
	for {
		select {
		case <-tick:
			fmt.Println("tick 1 second")
		case <-after:
			fmt.Println("after 3 second")
			return
		case value := <- channel:
			fmt.Println("got", value)
		default:
			fmt.Println("come into default")
			time.Sleep(500 * time.Millisecond)
		}
	}
}
~~~

输出结果

~~~
.
.
.
.
got 0
got 0
got 0
got 0
got 0
after 3 second
~~~

简直是车祸现场，幸好设置了3s主动退出，那case的时候，有没有办法判断这个channel是否关闭了呢，当然是可以的，看下面的程序

~~~go
func example() {
	tick := time.Tick(time.Second)
	after := time.After(3 * time.Second)
	channel := make(chan int, 1)
	go func() {
		channel <- 1
		close(channel)
	}()
	for {
		select {
		case <-tick:
			fmt.Println("tick 1 second")
		case <-after:
			fmt.Println("after 3 second")
			return
		case value, ok := <- channel:
			if ok {
				fmt.Println("got", value)
			} else {
				fmt.Println("channel is closed")
				time.Sleep(time.Second)
			}
		default:
			fmt.Println("come into default")
			time.Sleep(500 * time.Millisecond)
		}
	}
}
~~~

输出结果

~~~
come into default
got 1
channel is closed
tick 1 second
channel is closed
channel is closed
after 3 second
~~~

综上可以看出，通过 `value, ok := <- channel` 这种形式，ok获取的就是用来判断channel

是否关闭的，ok为 true，表示channel正常，否则，channel就是关闭的