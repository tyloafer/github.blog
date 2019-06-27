---
title: Sync.Pool浅析
tags:
  - Go
categories:
  - Go
date: '2019-05-19 20:45'
abbrlink: 61660
---

sync pool使用来存放临时变量的一个缓冲区，但是这个缓冲区并不可靠，每次gc的时候，都会首先清除缓冲区，所以，假如一个slice仅仅存放在 Pool 中，而没有其他地方引用，则会被当成垃圾清理掉。

<!--more-->

## 概念

> A Pool is a set of temporary objects that may be individually saved and
> retrieved.
>
> Any item stored in the Pool may be removed automatically at any time without
> notification. If the Pool holds the only reference when this happens, the
> item might be deallocated.
>
> A Pool is safe for use by multiple goroutines simultaneously.
>
> Pool's purpose is to cache allocated but unused items for later reuse,
> relieving pressure on the garbage collector. That is, it makes it easy to
> build efficient, thread-safe free lists. However, it is not suitable for all
> free lists.
>
> An appropriate use of a Pool is to manage a group of temporary items
> silently shared among and potentially reused by concurrent independent
> clients of a package. Pool provides a way to amortize allocation overhead
> across many clients.
>
> An example of good use of a Pool is in the fmt package, which maintains a
> dynamically-sized store of temporary output buffers. The store scales under
> load (when many goroutines are actively printing) and shrinks when
> quiescent.

## 图示

![](http://note-1253518569.cossh.myqcloud.com/20190519203029.png)

sync.pool的结构组成如上图所示，在这里可能会有两个问题

1. 我们实例化 Sync.Pool的时候，为什么实例化了一个LocalPool数组，怎么确定我的数据应该存储在LocalPool数组的哪个单元？
2. PoolLocalInternal 里面的成员有private和shared，为什么要做这两种区分？

## 源码分析

### Put

#### Put()

```
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	// race检测 先忽略这一块
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
	
	// 根据自身的goroutine的id，获取对应的PoolLocal的地址，后面具体分析
	l := p.pin()
	// 如果private字段为空的话，首先给private字段赋值
	if l.private == nil {
		l.private = x
		x = nil
	}
	runtime_procUnpin()
	// 如果private字段, 则添加到shared字段，因为 shared字段可以被其他goroutine获取，所以这里需要加锁
	if x != nil {
		l.Lock()
		l.shared = append(l.shared, x)
		l.Unlock()
	}
	if race.Enabled {
		race.Enable()
	}
}
```

#### pin()

```
func (p *Pool) pin() *poolLocal {
   // 获取当前的Pid/P，数量由
	pid := runtime_procPin()
	// LocalPool的数量
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
	// 如果获取到的pid比LocalPool数组的长度小，返回对应的LocalPool
	if uintptr(pid) < s {
		return indexLocal(l, pid)
	}
	// 如果pid比LocalPool数组的长度大，进一步确认，这个函数后面讨论
	return p.pinSlow()
}
```

`runtime_procPin()` 这个是获取当前运行的pid，具体实现没有查到， 但是`runtime_procPin()`返回的数值范围是由 `runtime.GOMAXPROCS(0)` 决定的。网上有篇文章可以参考一下[《Golang 的 协程调度机制 与 GOMAXPROCS 性能调优》](https://juejin.im/post/5b7678f451882533110e8948)，这里暂不深入

#### PinSlow()

```
func (p *Pool) pinSlow() *poolLocal {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	// 再次检查是否LocalPool是否有对应的索引，避免其他的线程造成影响
	s := p.localSize
	l := p.local
	if uintptr(pid) < s {
		return indexLocal(l, pid)
	}
	// 如果local为nil，说明是新构建的Pool结构体，加紧allPools slice里面
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	// 重新获取 GOMAXPROCS，并根据这个设置PoolLocal的大小
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
	// 找到当前goroutine对应的地址，并返回
	return &local[pid]
}
```

#### Put 逻辑

综上，Put的基本操作逻辑就是

- 获取当前执行的`Pid`
- 根据Pid，找到对应的`PoolLocal`，接着使用里面`PoolLocalInternal`
- 优先存入 `PoolLocalInternal` 的 `private`属性，其次粗如 `PoolLocalInternal` 的 `shared` 这个slice里面

### Get

#### Get()

```
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
	
	// 获取到LocalPool
	l := p.pin()
	
	// 把private数据值拷贝一份，然后把private设置为nil，因为如果private有数据，把private数据返回后，要把private设置为nil，如果private没有数据，则原先就是nil，添加这一步也没有关系
	x := l.private
	l.private = nil
	runtime_procUnpin()
	// 如果private里面没有数据，则从shared里面去找
	if x == nil {
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
		}
		l.Unlock()
		// 如果当前线程下对应的LocalPool，没有数据，则调用getSlow()，从其他的LocalPool的shared里面获取数据，后面解析 getSlow
		if x == nil {
			x = p.getSlow()
		}
	}
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
	// 如果 从private shared及其他的LocalPool的shared里面都获取不到数据，且注册的New函数不为空，则执行注册的New函数
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

#### getSlow()

```
func (p *Pool) getSlow() (x interface{}) {
	// See the comment in pin regarding ordering of the loads.
	// 获取LocalPool的size
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	local := p.local                         // load-consume
	// Try to steal one element from other procs.
	pid := runtime_procPin()
	runtime_procUnpin()
	// 便利LocalPool，获取shared里面的数据，找到就返回
	for i := 0; i < int(size); i++ {
		l := indexLocal(local, (pid+i+1)%int(size))
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
			l.Unlock()
			break
		}
		l.Unlock()
	}
	return x
}
```

通过上面的逻辑可以看出，`shared` 里面的数据是会被其他的P检索到的，而 `private`里面的数据是不会的，所以在获取`shared`里面数据的时候，需要加锁

### poolCleanup

这个函数是Pool包里面提供的，用来清理Pool的，但是官方的实现略显粗暴

```
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.
	// Defensively zero out everything, 2 reasons:
	// 1. To prevent false retention of whole Pools.
	// 2. If GC happens while a goroutine works with l.shared in Put/Get,
	//    it will retain whole Pool. So next cycle memory consumption would be doubled.
	// 便利所有的Sync.Pool
	for i, p := range allPools {
		allPools[i] = nil
		// 遍历Pool里面的LocalPool，并清空里面的数据
		for i := 0; i < int(p.localSize); i++ {
			l := indexLocal(p.local, i)
			l.private = nil
			for j := range l.shared {
				l.shared[j] = nil
			}
			l.shared = nil
		}
		p.local = nil
		p.localSize = 0
	}
	// 清空allPools
	allPools = []*Pool{}
}
```

这个函数会在GC之前调用，这也就解释了官方的下面一句话

> Any item stored in the Pool may be removed automatically at any time without
> notification. If the Pool holds the only reference when this happens, the
> item might be deallocated.

如果一个数据仅仅在Pool中有引用，那么就需要担心这个数据被GC清理掉

### 问题分析

针对于上面提出的两个问题，做一下简单的分析

> 我们实例化 Sync.Pool的时候，为什么实例化了一个LocalPool数组，怎么确定我的数据应该存储在LocalPool数组的哪个单元？

这里的LocalPool是根据不同的pid来区分的，保证private数据的线程安全，程序运行的时候可以获取到pid，然后使用pid作为LocalPool的索引，找到对应的地址即可

> PoolLocalInternal 里面的成员有private和shared，为什么要做这两种区分？

`private` 是 P 专属的， `shared`是可以被其他的P获取到的



## 参考文档

[《sync.Pool源码实现》](http://www.mckee.cn/golang/pkg/syncpool)
《Go 语言学习笔记 - 雨痕》