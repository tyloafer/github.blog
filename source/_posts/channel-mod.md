---
title: Go学习之Channel的一些模式
tags:
  - Go
categories:
  - Go
date: '2019-06-19 18:04'
abbrlink: 30113
---

除了在goroutine之间安全的传递数据之外，在看了《Concurrency in Go》之后，感慨channel还有那么多模式可供使用，在个人的学习中总结了以下几种常用的模式

<!--more-->

# pipeline

## 概念

我们以爬虫为例，一般爬虫分为如下步骤：

抓取页面 -> 解析页面 -> 整合数据分析 -> 分析结果入库

如果你把上面所有的步骤都放在一个函数里面处理，那会是多难看，多难以维护，从解耦角度考虑，我们可以起四个进程，分别承担不同的角色，例如，进程1负责抓取页面， 进程2负责解析页面，等等，各个进程拿到一个数据后，交给下一个进程来处理，这就是pipeline的基本思想，每个角色只负责关心自己的东西

## 示例

给定一个数n，执行 (n*2 + 1) * 2的操作

~~~go
func pipeline() {
	generator := func(done chan interface{}, intergers ...int) <-chan int {
		inStream := make(chan int)
		go func() {
			defer close(inStream)
			for _, i := range intergers {
				select {
				case <-done:
					return
				case inStream <- i:
				}
			}
		}()
		return inStream
	}

	add := func(done <-chan interface{}, inStream <-chan int, increment int) <-chan int {
		addInStream := make(chan int)
		go func() {
			defer close(addInStream)
			for i := range inStream {
				select {
				case <-done:
					return
				case addInStream <- i + increment:
				}
			}
		}()
		return addInStream
	}

	multiply := func(done <-chan interface{}, inStream <-chan int, increment int) <-chan int {
		multiplyInStream := make(chan int)
		go func() {
			defer close(multiplyInStream)
			for i := range inStream {
				select {
				case <-done:
					return
				case multiplyInStream <- i * increment:
				}
			}
		}()
		return multiplyInStream
	}

	done := make(chan interface{})
	defer close(done)
	inStream := generator(done, []int{1, 2, 3, 4, 5, 6, 7}...)
	pipeline := multiply(done, add(done, multiply(done, inStream, 2), 1), 2)

	for v := range pipeline {
		fmt.Println(v)
	}
}

~~~



# 扇入扇出

在pipeline模型中，是一种高效的流式处理，但是假如pipeline中有a,b,c三个环节，b环节处理的特别慢，这时候就会影响到c环节的处理，如果增加b环节进程处理的数量，也就可以减弱b环节的慢处理对整个pipeline的影响，那么a->多个b的过程就是 **扇入**， 多个b环节输出数据到c环节，就是**扇出**

## 示例

~~~go
func FanInFanOut() {
	producer := func(intergers ...int) <-chan interface{} {
		inStream := make(chan interface{})
		go func() {
			defer close(inStream)
			for _, v := range intergers {
				time.Sleep(5 * time.Second)
				inStream <- v
			}
		}()
		return inStream
	}

	fanIn := func(channels ...<-chan interface{},
	) <-chan interface{} {
		var wg sync.WaitGroup
		multiplexStream := make(chan interface{})

		multiplex := func(c <-chan interface{}) {
			defer wg.Done()
			for i := range c {
				multiplexStream <- i
			}
		}

		wg.Add(len(channels))
		for _, c := range channels {
			go multiplex(c)
		}
		go func() {
			wg.Wait()
			close(multiplexStream)
		}()
		return multiplexStream
	}

	consumer := func(inStream <-chan interface{}) {
		for v := range inStream {
			fmt.Println(v)
		}
	}

	nums := runtime.NumCPU()
	producerStreams := make([]<-chan interface{}, nums)
	for i := 0; i < nums; i++ {
		producerStreams[i] = producer(i)
	}

	consumer(fanIn(producerStreams...))
}
~~~



# tee- channel

## 概念

假如你从channel中拿到了一条sql语句，这时候，你想对这条sql记录，分析并执行，那你就需要将这条sql分别转发给这三个任务对应的channel，tee-channel 就是做这个事情的

## 示例

~~~go
func teeChannel() {
	producer := func(intergers ...int) <-chan interface{} {
		inStream := make(chan interface{})
		go func() {
			defer close(inStream)
			for _, v := range intergers {
				inStream <- v
			}
		}()
		return inStream
	}
	tee := func(in <-chan interface{}) (_, _ <-chan interface{}) {
		out1 := make(chan interface{})
		out2 := make(chan interface{})
		go func() {
			defer close(out1)
			defer close(out2)

			for val := range in {
				out1, out2 := out1, out2
				for i := 0; i < 2; i++ {
					select {
					case out1 <- val:
						out1 = nil
					case out2 <- val:
						out2 = nil
					}
				}
			}
		}()
		return out1, out2
	}

	out1, out2 := tee(producer(1, 2, 3, 4, 5))
	for val1 := range out1 {
		fmt.Printf("out1: %v, out2: %v", val1, <-out2)
	}
}
~~~



# 桥接channel

## 概念

无论是前面提到的**pipeline**还是**扇入扇出**，每个goroutine都是对一个channel进行消费，但是实际场景中，可能会有多个channel来供给我们消费，而作为消费者，我们不关心这些值是来自于哪个channel，这种情况下，处理一个充满channel的channel可能会很多。如果我们定义一个功能，可以将充满channel的channel拆解为一个简单的channel，这将使消费者更专注于手头的工作，这就是**桥接channel**的思想

## 示例

~~~go
func bridge() {
	gen := func() <-chan <-chan interface{} {
		in := make(chan (<-chan interface{}))
		go func() {
			defer close(in)
			for i := 0; i < 10; i++ {
				stream := make(chan interface{}, 1)
				stream <- i
				close(stream)
				in <- stream
			}
		}()
		return in
	}

	bridge := func(in <-chan (<-chan interface{})) <-chan interface{} {
		valStream := make(chan interface{})
		go func() {
			defer close(valStream)
			for {
				stream := make(<-chan interface{})
				select {
				case maybeStream, ok := <-in:
					if ok == false {
						return
					}
					stream = maybeStream
				}
				for val := range stream {
					valStream <- val
				}
			}
		}()
		return valStream
	}

	for val := range bridge(gen()) {
		fmt.Println(val)
	}
}
~~~

