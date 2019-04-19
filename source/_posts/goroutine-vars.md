---
title: Golang常见错误之goroutine使用变量问题
tags:
  - Go
categories:
  - Go
date: '2019-04-18 18:45'
abbrlink: 22138
---

https://www.jb51.net/article/128539.htm
针对这篇文章引发的一些实验
<!-- more -->

在看这篇文章的时候，总感觉用文章中的例子说明值拷贝和指针拷贝是有问题的，而且go官方说，他们的拷贝只有值拷贝一种，那么为什么使用指针类型和非指针类型，会出现不同的现象，在追踪的过程了就发现了下面的奇葩实例
## 试验代码

    package main
    
    import (
    	"fmt"
    	"time"
    )
    
    type fieldNoPointer struct {
    	name string
    }
    
    func (p fieldNoPointer) Print() {
    	fmt.Println(p.name)
    }
    
    type fieldPointer struct {
    	name string
    }
    
    func (p *fieldPointer) Print() {
    	fmt.Println(p.name)
    }
    
    func main() {
    	printDataWithPointerUseFieldPointerWithFunc()
    	printDataWithPointerUseFieldPointerWithNoFunc()
    	printDataWithNoPointerUseFieldPointerWithFunc()
    	printDataWithNoPointerUseFieldPointerWithNoFunc()
    	printDataWithPointerUseFieldNoPointerWithFunc()
    	printDataWithPointerUseFieldNoPointerWithNoFunc()
    	printDataWithNoPointerUseFieldNoPointerWithFunc()
    	printDataWithNoPointerUseFieldNoPointerWithNoFunc()
    	time.Sleep(1)
    }
    
    func printDataWithPointerUseFieldPointerWithFunc()  {
    	fmt.Println("print by get printDataWithPointerUseFieldPointerWithFunc")
    	data := []*fieldPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go func() {
    			v.Print()
    		}()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithPointerUseFieldPointerWithNoFunc()  {
    	fmt.Println("print by get printDataWithPointerUseFieldPointerWithNoFunc")
    	data := []*fieldPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go v.Print()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithNoPointerUseFieldPointerWithFunc()  {
    	fmt.Println("print by get printDataWithNoPointerUseFieldPointerWithFunc")
    	data := []fieldPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go func() {
    			v.Print()
    		}()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithNoPointerUseFieldPointerWithNoFunc()  {
    	fmt.Println("print by get printDataWithNoPointerUseFieldPointerWithNoFunc")
    	data := []fieldPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go v.Print()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithPointerUseFieldNoPointerWithFunc()  {
    	fmt.Println("print by get printDataWithPointerUseFieldNoPointerWithFunc")
    	data := []*fieldNoPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go func() {
    			v.Print()
    		}()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithPointerUseFieldNoPointerWithNoFunc()  {
    	fmt.Println("print by get printDataWithPointerUseFieldNoPointerWithNoFunc")
    	data := []*fieldNoPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go v.Print()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithNoPointerUseFieldNoPointerWithFunc()  {
    	fmt.Println("print by get printDataWithNoPointerUseFieldNoPointerWithFunc")
    	data := []fieldNoPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go func() {
    			v.Print()
    		}()
    	}
    	time.Sleep(time.Second)
    }
    
    func printDataWithNoPointerUseFieldNoPointerWithNoFunc()  {
    	fmt.Println("print by get printDataWithNoPointerUseFieldNoPointerWithNoFunc")
    	data := []fieldNoPointer{{"one"},{"two"},{"three"}}
    
    	for _,v := range data {
    		go v.Print()
    	}
    	time.Sleep(time.Second)
    }

## 实验结果

    print by get printDataWithPointerUseFieldPointerWithFunc
    three
    three
    three
    print by get printDataWithPointerUseFieldPointerWithNoFunc
    one
    two
    three
    print by get printDataWithNoPointerUseFieldPointerWithFunc
    three
    three
    three
    print by get printDataWithNoPointerUseFieldPointerWithNoFunc
    three
    three
    three
    print by get printDataWithPointerUseFieldNoPointerWithFunc
    three
    three
    three
    print by get printDataWithPointerUseFieldNoPointerWithNoFunc
    three
    one
    two
    print by get printDataWithNoPointerUseFieldNoPointerWithFunc
    three
    three
    three
    print by get printDataWithNoPointerUseFieldNoPointerWithNoFunc
    three
    one
    two

在这里，我以 我以 `struct的func是否指针类型` `struc的数据是否是指针类型` 以及 `goroutine 是否使用匿名函数` 为例作为测试的，上面的测试例子都是无意中发现的

图表


|                       | Goroutine func 且 struct数据是指针类型 | Goroutine且 struct数据是指针类型 | Goroutine func 且 struct数据不是指针类型 | Goroutine且 struct数据不是指针类型 |
| --------------------- | -------------------------------------- | -------------------------------- | ---------------------------------------- | ---------------------------------- |
| Struct func指针类型   | 异常                                   | 正常                             | 异常                                     | 异常                               |
| Struct func非指针类型 | 异常                                   | 异常                             | 异常                                     | 正常                               |

## 结论
在goroutine里面使用变量的时候，需要传递，上面案例中不传递变量直接使用的方法都是错误的