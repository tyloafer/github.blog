---
title: 深入理解Go-runtime.SetFinalizer原理剖析
tags:
  - Go
categories:
  - Go
date: '2019-09-06 21:04'
abbrlink: 23555
---

finalizer是与对象关联的一个函数，通过`runtime.SetFinalizer` 来设置，它在对象被GC的时候，这个finalizer会被调用，以完成对象生命中最后一程。由于finalizer的存在，导致了对象在三色标记中，不可能被标为白色对象，也就是垃圾，所以，这个对象的生命也会得以延续一个GC周期。正如defer一样，我们也可以通过 Finalizer 完成一些类似于资源释放的操作

<!--more-->

# 结构概览

## heap

~~~go
type mspan struct {
	// 当前span上所有对象的special串成链表
	// special中有个offset，就是数据对象在span上的offset，通过offset，将数据对象和special关联起来
	specials    *special   // linked list of special records sorted by offset.
}
~~~

## special

~~~go
type special struct {
	next   *special // linked list in span
	// 数据对象在span上的offset
	offset uint16   // span offset of object
	kind   byte     // kind of special
}
~~~

##specialfinalizer


```go
type specialfinalizer struct {
	special special
	fn      *funcval // May be a heap pointer.
	// return的数据的大小
	nret    uintptr
	// 第一个参数的类型
	fint    *_type   // May be a heap pointer, but always live.
	// 与finalizer关联的数据对象的指针类型
	ot      *ptrtype // May be a heap pointer, but always live.
}
```

## finalizer

```go
type finalizer struct {
	fn   *funcval       // function to call (may be a heap pointer)
	arg  unsafe.Pointer // ptr to object (may be a heap pointer)
	nret uintptr        // bytes of return values from fn
	fint *_type         // type of first argument of fn
	ot   *ptrtype       // type of ptr to object (may be a heap pointer)
}
```

## 全局变量

~~~go
var finlock mutex  // protects the following variables
// 运行finalizer的g，只有一个g，不用的时候休眠，需要的时候再唤醒
var fing *g        // goroutine that runs finalizers
// finalizer的全局队列，这里是已经设置的finalizer串成的链表
var finq *finblock // list of finalizers that are to be executed
// 已经释放的finblock的链表，用finc缓存起来，以后需要使用的时候可以直接取走，避免再走一遍内存分配了
var finc *finblock // cache of free blocks
var finptrmask [_FinBlockSize / sys.PtrSize / 8]byte
var fingwait bool  // fing的标志位，通过 fingwait和fingwake，来确定是否需要唤醒fing
var fingwake bool
// 所有的blocks串成的链表
var allfin *finblock // list of all blocks
~~~

# 源码分析

## 创建finalizer

### main

```go
func main() {
	// i 就是后面说的 数据对象
	var i = 3
	// 这里的func 就是后面一直说的 finalizer
	runtime.SetFinalizer(&i, func(i *int) {
		fmt.Println(i, *i, "set finalizer")
	})
	time.Sleep(time.Second * 5)
}
```

### SetFinalizer

根据 数据对象 ，生成一个special对象，并绑定到 数据对象 所在的span，串联到span.specials上，并且确保fing的存在

```go
func SetFinalizer(obj interface{}, finalizer interface{}) {
	if debug.sbrk != 0 {
		// debug.sbrk never frees memory, so no finalizers run
		// (and we don't have the data structures to record them).
		return
	}
	e := efaceOf(&obj)
	etyp := e._type
	// ---- 省略数据校验的逻辑 ---
	ot := (*ptrtype)(unsafe.Pointer(etyp))

	// find the containing object
	// 在内存中找不到分配的地址时 base==0，setFinalizer 是在内存回收的时候调用，没有分配就不会回收
	base, _, _ := findObject(uintptr(e.data), 0, 0)

	f := efaceOf(&finalizer)
	ftyp := f._type
	// 如果 finalizer type == nil，尝试移除（没有的话，就不需要移除了）
	if ftyp == nil {
		// switch to system stack and remove finalizer
		systemstack(func() {
			removefinalizer(e.data)
		})
		return
	}
	// --- 对finalizer参数数量及类型进行校验 --
	if ftyp.kind&kindMask != kindFunc {
		throw("runtime.SetFinalizer: second argument is " + ftyp.string() + ", not a function")
	}
	ft := (*functype)(unsafe.Pointer(ftyp))
	if ft.dotdotdot() {
		throw("runtime.SetFinalizer: cannot pass " + etyp.string() + " to finalizer " + ftyp.string() + " because dotdotdot")
	}
	if ft.inCount != 1 {
		throw("runtime.SetFinalizer: cannot pass " + etyp.string() + " to finalizer " + ftyp.string())
	}
	fint := ft.in()[0]
	switch {
	case fint == etyp:
		// ok - same type
		goto okarg
	case fint.kind&kindMask == kindPtr:
		if (fint.uncommon() == nil || etyp.uncommon() == nil) && (*ptrtype)(unsafe.Pointer(fint)).elem == ot.elem {
			// ok - not same type, but both pointers,
			// one or the other is unnamed, and same element type, so assignable.
			goto okarg
		}
	case fint.kind&kindMask == kindInterface:
		ityp := (*interfacetype)(unsafe.Pointer(fint))
		if len(ityp.mhdr) == 0 {
			// ok - satisfies empty interface
			goto okarg
		}
		if _, ok := assertE2I2(ityp, *efaceOf(&obj)); ok {
			goto okarg
		}
	}
	throw("runtime.SetFinalizer: cannot pass " + etyp.string() + " to finalizer " + ftyp.string())
okarg:
	// compute size needed for return parameters
	// 计算返回参数的大小并进行对齐
	nret := uintptr(0)
	for _, t := range ft.out() {
		nret = round(nret, uintptr(t.align)) + uintptr(t.size)
	}
	nret = round(nret, sys.PtrSize)

	// make sure we have a finalizer goroutine
	// 确保 finalizer 有一个 goroutine
	createfing()

	systemstack(func() {
		// 却换到g0，添加finalizer，并且不能重复设置
		if !addfinalizer(e.data, (*funcval)(f.data), nret, fint, ot) {
			throw("runtime.SetFinalizer: finalizer already set")
		}
	})
}
```

这里逻辑没什么复杂的，只是在参数、类型的判断等上面，比较的麻烦

### removefinalizer

通过removespecial，找到数据对象p所对应的special对象，如果找到的话，释放mheap上对应的内存

```go
func removefinalizer(p unsafe.Pointer) {
	// 根据数据p找到对应的special对象
	s := (*specialfinalizer)(unsafe.Pointer(removespecial(p, _KindSpecialFinalizer)))
	if s == nil {
		return // there wasn't a finalizer to remove
	}
	lock(&mheap_.speciallock)
	// 释放找到的special所对应的内存
	mheap_.specialfinalizeralloc.free(unsafe.Pointer(s))
	unlock(&mheap_.speciallock)
}
```

这里的函数，虽然叫removefinalizer， 但是这里暂时跟finalizer结构体没有关系，都是在跟special结构体打交道，后面的addfinalizer也是一样的

### removespecial

遍历数据所在的span的specials，如果找到了指定数据p的special的话，就从specials中移除，并返回

```go
func removespecial(p unsafe.Pointer, kind uint8) *special {
	// 找到数据p所在的span
	span := spanOfHeap(uintptr(p))
	if span == nil {
		throw("removespecial on invalid pointer")
	}

	// Ensure that the span is swept.
	// Sweeping accesses the specials list w/o locks, so we have
	// to synchronize with it. And it's just much safer.
	mp := acquirem()
	// 保证span被清扫过了
	span.ensureSwept()
	// 获取数据p的偏移量，根据偏移量去寻找p对应的special
	offset := uintptr(p) - span.base()

	lock(&span.speciallock)
	t := &span.specials
	// 遍历span.specials这个链表
	for {
		s := *t
		if s == nil {
			break
		}
		// This function is used for finalizers only, so we don't check for
		// "interior" specials (p must be exactly equal to s->offset).
		if offset == uintptr(s.offset) && kind == s.kind {
			// 找到了，修改指针，将当前找到的special移除
			*t = s.next
			unlock(&span.speciallock)
			releasem(mp)
			return s
		}
		t = &s.next
	}
	unlock(&span.speciallock)
	releasem(mp)
	// 没有找到，就返回nil
	return nil
}
```

### addfinalizer

正好跟removefinalizer相反，这个就是根据数据对象p，创建对应的special，然后添加到span.specials链表上面

```go
func addfinalizer(p unsafe.Pointer, f *funcval, nret uintptr, fint *_type, ot *ptrtype) bool {
	lock(&mheap_.speciallock)
	// 分配出来一块内存供finalizer使用
	s := (*specialfinalizer)(mheap_.specialfinalizeralloc.alloc())
	unlock(&mheap_.speciallock)
	s.special.kind = _KindSpecialFinalizer
	s.fn = f
	s.nret = nret
	s.fint = fint
	s.ot = ot
	if addspecial(p, &s.special) {

		return true
	}

	// There was an old finalizer
	// 没有添加成功，是因为p已经有了一个special对象了
	lock(&mheap_.speciallock)
	mheap_.specialfinalizeralloc.free(unsafe.Pointer(s))
	unlock(&mheap_.speciallock)
	return false
}
```

### addspecial

这里是添加special的主逻辑

```go
func addspecial(p unsafe.Pointer, s *special) bool {
	span := spanOfHeap(uintptr(p))
	if span == nil {
		throw("addspecial on invalid pointer")
	}
	// 同 removerspecial一样，确保这个span已经清扫过了
	mp := acquirem()
	span.ensureSwept()

	offset := uintptr(p) - span.base()
	kind := s.kind

	lock(&span.speciallock)

	// Find splice point, check for existing record.
	t := &span.specials
	for {
		x := *t
		if x == nil {
			break
		}
		if offset == uintptr(x.offset) && kind == x.kind {
			// 已经存在了，不能在增加了，一个数据对象，只能绑定一个finalizer
			unlock(&span.speciallock)
			releasem(mp)
			return false // already exists
		}
		if offset < uintptr(x.offset) || (offset == uintptr(x.offset) && kind < x.kind) {
			break
		}
		t = &x.next
	}

	// Splice in record, fill in offset.
	// 添加到 specials 队列尾
	s.offset = uint16(offset)
	s.next = *t
	*t = s
	unlock(&span.speciallock)
	releasem(mp)

	return true
}
```

### createfing

这个函数是保证，创建了finalizer之后，有一个goroutine去运行，这里只运行一次，这个goroutine会由全局变量 fing 记录

```go
func createfing() {
	// start the finalizer goroutine exactly once
	// 进创建一个goroutine，进行时刻监控运行
	if fingCreate == 0 && atomic.Cas(&fingCreate, 0, 1) {
		// 开启一个goroutine运行
		go runfinq()
	}
}
```

## 执行finalizer

在上面的 `createfing` 的会尝试创建一个goroutine去执行，接下来就分析一下执行流程吧

```go
func runfinq() {
	var (
		frame    unsafe.Pointer
		framecap uintptr
	)

	for {
		lock(&finlock)
		// 获取finq 全局队列，并清空全局队列
		fb := finq
		finq = nil
		if fb == nil {
			// 如果全局队列为空，休眠当前g，等待被唤醒
			gp := getg()
			fing = gp
			// 设置fing的状态标志位
			fingwait = true
			goparkunlock(&finlock, waitReasonFinalizerWait, traceEvGoBlock, 1)
			continue
		}
		unlock(&finlock)
		// 循环执行runq链表里的fin数组
		for fb != nil {
			for i := fb.cnt; i > 0; i-- {
				f := &fb.fin[i-1]
				// 获取存储当前finalizer的返回数据的大小，如果比之前大，则分配
				framesz := unsafe.Sizeof((interface{})(nil)) + f.nret
				if framecap < framesz {
					// The frame does not contain pointers interesting for GC,
					// all not yet finalized objects are stored in finq.
					// If we do not mark it as FlagNoScan,
					// the last finalized object is not collected.
					frame = mallocgc(framesz, nil, true)
					framecap = framesz
				}

				if f.fint == nil {
					throw("missing type in runfinq")
				}
				// frame is effectively uninitialized
				// memory. That means we have to clear
				// it before writing to it to avoid
				// confusing the write barrier.
				// 清空frame内存存储
				*(*[2]uintptr)(frame) = [2]uintptr{}
				switch f.fint.kind & kindMask {
				case kindPtr:
					// direct use of pointer
					*(*unsafe.Pointer)(frame) = f.arg
				case kindInterface:
					ityp := (*interfacetype)(unsafe.Pointer(f.fint))
					// set up with empty interface
					(*eface)(frame)._type = &f.ot.typ
					(*eface)(frame).data = f.arg
					if len(ityp.mhdr) != 0 {
						// convert to interface with methods
						// this conversion is guaranteed to succeed - we checked in SetFinalizer
						*(*iface)(frame) = assertE2I(ityp, *(*eface)(frame))
					}
				default:
					throw("bad kind in runfinq")
				}
				// 调用finalizer函数
				fingRunning = true
				reflectcall(nil, unsafe.Pointer(f.fn), frame, uint32(framesz), uint32(framesz))
				fingRunning = false

				// Drop finalizer queue heap references
				// before hiding them from markroot.
				// This also ensures these will be
				// clear if we reuse the finalizer.
				// 清空finalizer的属性
				f.fn = nil
				f.arg = nil
				f.ot = nil
				atomic.Store(&fb.cnt, i-1)
			}
			// 将已经完成的finalizer放入finc以作缓存，避免再次分配内存
			next := fb.next
			lock(&finlock)
			fb.next = finc
			finc = fb
			unlock(&finlock)
			fb = next
		}
	}
}
```

看完上面的流程的时候，突然发现有点懵逼

1. 全局队列finq中是什么时候被插入数据 finalizer的？
2. g如果休眠了，那怎么被唤醒呢？

先针对第一个问题分析：

插入队列的操作，要追溯到我们之前分析的GC  [深入理解Go-垃圾回收机制](https://segmentfault.com/a/1190000020086769) 了，在`sweep` 中有下面一段函数

### sweep

~~~go
func (s *mspan) sweep(preserve bool) bool {
	....
	specialp := &s.specials
	special := *specialp
	for special != nil {
		....
		if special.kind == _KindSpecialFinalizer || !hasFin {
			// Splice out special record.
			y := special
			special = special.next
			*specialp = special
			// 加入全局finq队列的入口就在这里了
			freespecial(y, unsafe.Pointer(p), size)
		}
		....
	}
	....
}
~~~

### freespecial

在gc的时候，不仅要把special对应的内存释放掉，而且把specials整理创建对应dinalizer对象，并插入到 finq队列里面

```go
func freespecial(s *special, p unsafe.Pointer, size uintptr) {
	switch s.kind {
	case _KindSpecialFinalizer:
		// 把这个finalizer加入到全局队列
		sf := (*specialfinalizer)(unsafe.Pointer(s))
		queuefinalizer(p, sf.fn, sf.nret, sf.fint, sf.ot)
		lock(&mheap_.speciallock)
		mheap_.specialfinalizeralloc.free(unsafe.Pointer(sf))
		unlock(&mheap_.speciallock)
	// 下面两种情况不在分析范围内，省略
	case _KindSpecialProfile:
		sp := (*specialprofile)(unsafe.Pointer(s))
		mProf_Free(sp.b, size)
		lock(&mheap_.speciallock)
		mheap_.specialprofilealloc.free(unsafe.Pointer(sp))
		unlock(&mheap_.speciallock)
	default:
		throw("bad special kind")
		panic("not reached")
	}
}
```

### queuefinalizer

```go
func queuefinalizer(p unsafe.Pointer, fn *funcval, nret uintptr, fint *_type, ot *ptrtype) {
	lock(&finlock)
	// 如果finq为空或finq的内部数组已经满了，则从finc或重新分配 来获取block并插入到finq的链表头
	if finq == nil || finq.cnt == uint32(len(finq.fin)) {
		if finc == nil {
			finc = (*finblock)(persistentalloc(_FinBlockSize, 0, &memstats.gc_sys))
			finc.alllink = allfin
			allfin = finc
			if finptrmask[0] == 0 {
				// Build pointer mask for Finalizer array in block.
				// Check assumptions made in finalizer1 array above.
				if (unsafe.Sizeof(finalizer{}) != 5*sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.fn) != 0 ||
					unsafe.Offsetof(finalizer{}.arg) != sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.nret) != 2*sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.fint) != 3*sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.ot) != 4*sys.PtrSize) {
					throw("finalizer out of sync")
				}
				for i := range finptrmask {
					finptrmask[i] = finalizer1[i%len(finalizer1)]
				}
			}
		}
		// 从finc中移除并获取链表头
		block := finc
		finc = block.next
		// 将从finc获取到的链表挂载到finq的队列头，finq指向新的block
		block.next = finq
		finq = block
	}
	// 根据finq.cnt获取索引对应的block
	f := &finq.fin[finq.cnt]
	atomic.Xadd(&finq.cnt, +1) // Sync with markroots
	// 设置相关属性
	f.fn = fn
	f.nret = nret
	f.fint = fint
	f.ot = ot
	f.arg = p
	// 设置唤醒标志
	fingwake = true
	unlock(&finlock)
}
```

至此，也就明白了，runq全局队列是怎么被填充的了

那么，第二个问题，当fing被休眠后，怎么被唤醒呢？

这里就需要追溯到，[深入理解Go-goroutine的实现及Scheduler分析](https://segmentfault.com/a/1190000020254937) 这篇文章了

### findrunnable

在 findrunnable 中有一段代码如下：

~~~go
func findrunnable() (gp *g, inheritTime bool) {
	// 通过状态位判断是否需要唤醒 fing， 通过wakefing来判断并返回fing
	if fingwait && fingwake {
		if gp := wakefing(); gp != nil {
			// 唤醒g，并从休眠出继续执行
			ready(gp, 0, true)
		}
	}
}
~~~

### wakefing

这里不仅会对状态位 fingwait fingwake做二次判断，而且，如果状态位符合唤醒要求的话，需要重置两个状态位

~~~go
func wakefing() *g {
	var res *g
	lock(&finlock)
	if fingwait && fingwake {
		fingwait = false
		fingwake = false
		res = fing
	}
	unlock(&finlock)
	return res
}
~~~

# 参考文档

- 《Go语言学习笔记》--雨痕