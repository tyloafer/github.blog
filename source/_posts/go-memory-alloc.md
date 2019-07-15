---
title: 深入理解Go-内存分配
tags:
  - Go
categories:
  - Go
date: '2019-07-14 20:04'
abbrlink: 19281
---

Go语言内置运行时（就是runtime），抛弃了传统的内存分配方式，改为自主管理，最开始是基于tcmalloc，虽然后面改动相对已经很大了。使用自主管理可以实现更好的内存使用模式，比如内存池、预分配等等，从而避免了系统调用所带来的性能问题。

<!--more-->

在了解Go的内存分配之前，我们可以看一下内存分配的基本策略，来帮助我们理解Go的内存分配

基本策略：

1. 每次从操作系统申请一大块内存，以减少系统调用
2. 将申请的大块内存按照特定大小预先切成小块，构成链表
3. 为对象分配内存时，从大小合适的链表中提取一块即可
4. 如果对象销毁，则将对象占用的内存，归还到原链表，以便复用
5. 如果限制内存过多，则尝试归还部分给操作系统，降低整体开销

下面我们从源码角度来分析Go的内存分配策略有何异同

# 准备

在追踪源码之前，我们需要首先了解一些概念和结构体

- span: 又多个地址连续的页（page）组成的大块内存
- object: 将span按特定大小切分成多个小块，每个小块可存储一个对象

## 对象分类

- 小对象（tiny）: size < 16byte
- 普通对象： 16byte ~ 32K
- 大对象（large）： size > 32K

## 大小转换

![](http://note-1253518569.cossh.myqcloud.com/20190715112737.png)

## 结构体

### mHeap

代表Go程序持有的所有堆空间，Go程序使用一个`mheap`的全局对象`_mheap`来管理堆内存。

~~~go
type mheap struct {
	lock      mutex
	free      [_MaxMHeapList]mSpanList // page在127以内的闲置的span列表
	freelarge mTreap                   // page数大于127的大span组成的树状结构体
	busy      [_MaxMHeapList]mSpanList // page在127以内的已分配的span列表
	busylarge mSpanList                // page数大于127的已分配的大span组成的列表

	// allspans is a slice of all mspans ever created. Each mspan
	// appears exactly once.
	// 所有创建过的mspan的slice
	allspans []*mspan // all spans out there

	// arenas is the heap arena map. It points to the metadata for
	// the heap for every arena frame of the entire usable virtual
	// address space.
	//
	// Use arenaIndex to compute indexes into this array.
	//
	// For regions of the address space that are not backed by the
	// Go heap, the arena map contains nil.
	//
	// Modifications are protected by mheap_.lock. Reads can be
	// performed without locking; however, a given entry can
	// transition from nil to non-nil at any time when the lock
	// isn't held. (Entries never transitions back to nil.)
	//
	// In general, this is a two-level mapping consisting of an L1
	// map and possibly many L2 maps. This saves space when there
	// are a huge number of arena frames. However, on many
	// platforms (even 64-bit), arenaL1Bits is 0, making this
	// effectively a single-level map. In this case, arenas[0]
	// will never be nil.
	// 一组heapArena组成，每一个heapArena都包含了连续的pagesPerArena个span，这个主要是为mheap管理span和垃圾回收服务，heapArena也有介绍
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	// heapArenaAlloc is pre-reserved space for allocating heapArena
	// objects. This is only used on 32-bit, where we pre-reserve
	// this space to avoid interleaving it with the heap itself.
	// 预先分配的 heapArena 对象的地址
	heapArenaAlloc linearAlloc

	// arenaHints is a list of addresses at which to attempt to
	// add more heap arenas. This is initially populated with a
	// set of general hint addresses, and grown with the bounds of
	// actual heap arena ranges.
	arenaHints *arenaHint

	// arena is a pre-reserved space for allocating heap arenas
	// (the actual arenas). This is only used on 32-bit.
	// 仅32位使用
	arena linearAlloc

	//_ uint32 // ensure 64-bit alignment of central

	// central free lists for small size classes.
	// the padding makes sure that the MCentrals are
	// spaced CacheLineSize bytes apart, so that each MCentral.lock
	// gets its own cache line.
	// central is indexed by spanClass.
	// mcentral 内存分配中心，mcache没有足够的内存分配的时候，会从mcentral分配
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
	}

	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	treapalloc            fixalloc // allocator for treapNodes* used by large objects
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
	arenaHintAlloc        fixalloc // allocator for arenaHints

	unused *specialfinalizer // never set, just here to force the specialfinalizer type into DWARF
}
~~~

### mSpanList

mSpan的链表，`free` `busy` `busyLarge` 上的mSpan都是通过链表串联起来的

~~~
type mSpanList struct {
	first *mspan // first span in list, or nil if none
	last  *mspan // last span in list, or nil if none
}
~~~

### mSpan

Go中内存管理的基本单元，是由一片连续的`8KB`的页组成的大块内存。注意，这里的页和操作系统本身的页并不是一回事，它一般是操作系统页大小的几倍。一句话概括：`mspan`是一个包含起始地址、`mspan`规格、页的数量等内容的双端链表。

~~~go
type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.

	startAddr uintptr // address of first byte of span aka s.base()
	// 该span锁包含的页数
	npages    uintptr // number of pages in span

	manualFreeList gclinkptr // list of free objects in _MSpanManual spans

	// freeindex is the slot index between 0 and nelems at which to begin scanning
	// for the next free object in this span.
	// Each allocation scans allocBits starting at freeindex until it encounters a 0
	// indicating a free object. freeindex is then adjusted so that subsequent scans begin
	// just past the newly discovered free object.
	//
	// If freeindex == nelem, this span has no free objects.
	//
	// allocBits is a bitmap of objects in this span.
	// If n >= freeindex and allocBits[n/8] & (1<<(n%8)) is 0
	// then object n is free;
	// otherwise, object n is allocated. Bits starting at nelem are
	// undefined and should never be referenced.
	//
	// Object n starts at address n*elemsize + (start << pageShift).
	// 用于定位下一个可用的object, 大小范围在 0- nelems 之间
	freeindex uintptr
	// TODO: Look up nelems from sizeclass and remove this field if it
	// helps performance.
	// span里object的数量
	nelems uintptr // number of object in the span.

	// Cache of the allocBits at freeindex. allocCache is shifted
	// such that the lowest bit corresponds to the bit freeindex.
	// allocCache holds the complement of allocBits, thus allowing
	// ctz (count trailing zero) to use it directly.
	// allocCache may contain bits beyond s.nelems; the caller must ignore
	// these.
	// 用于缓存freeindex开始的bitmap, 缓存的bit值与原值相反，ctz函数可以通过这个值快速计算出下一个 free object的index
	allocCache uint64

	// 分配位图，每一位代表每一块是否已经分配
	allocBits  *gcBits

	// 已经分配的object的数量
	allocCount  uint16     // number of allocated objects

	elemsize    uintptr    // computed from sizeclass or from npages

}
~~~

### spanClass

class表中的class ID，和Size Classs相关

~~~go
type spanClass uint8
~~~

### mTreap

这个结构是包含mspan的树状结构，主要是给 freeLarge使用，在查找对应classsize的大对象的时候，使用树状结构查找要比链表更快

~~~go
type mTreap struct {
	treap *treapNode
}
~~~

### mtreapNode

mTreap结构的节点，节点信息包含mspan和左右子节点等信息

~~~go
type treapNode struct {
	right     *treapNode // all treapNodes > this treap node
	left      *treapNode // all treapNodes < this treap node
	parent    *treapNode // direct parent of this node, nil if root
	npagesKey uintptr    // number of pages in spanKey, used as primary sort key
	spanKey   *mspan     // span of size npagesKey, used as secondary sort key
	priority  uint32     // random number used by treap algorithm to keep tree probabilistically balanced
}
~~~

### heapArena

heapArena存储的是arena的元数据， arenas是一组heapArena构成，所有的分配的内存都在 `arenas` 里面，大致 arenas[L1]\[L2] = heapArena， 而对于 分配出去的内存的 address，通过 `arenaIndex` 可以计算出 `L1 L2`， 从而找到该内存所对应的 arenas[L1]\[L2]，即 heapArena

~~~go
type heapArena struct {
	// bitmap stores the pointer/scalar bitmap for the words in
	// this arena. See mbitmap.go for a description. Use the
	// heapBits type to access this.
	bitmap [heapArenaBitmapBytes]byte

	// spans maps from virtual address page ID within this arena to *mspan.
	// For allocated spans, their pages map to the span itself.
	// For free spans, only the lowest and highest pages map to the span itself.
	// Internal pages map to an arbitrary span.
	// For pages that have never been allocated, spans entries are nil.
	//
	// Modifications are protected by mheap.lock. Reads can be
	// performed without locking, but ONLY from indexes that are
	// known to contain in-use or stack spans. This means there
	// must not be a safe-point between establishing that an
	// address is live and looking it up in the spans array.
	spans [pagesPerArena]*mspan
}
~~~

### arenaHint

这个是记录arena可以增长的地址

~~~go
type arenaHint struct {
	addr uintptr
	// down 为 true，表示可以扩展arena的大小
	down bool
	next *arenaHint
}
~~~

### mcentral

mcentral则是全局资源，为多个线程服务，当某个线程内存不足时会向mcentral申请，当某个线程释放内存时又会回收进mcentral

~~~go
type mcentral struct {
	lock      mutex
	spanclass spanClass
	// free object 的链表
	nonempty  mSpanList // list of spans with a free object, ie a nonempty free list
	// no free object 的链表
	empty     mSpanList // list of spans with no free objects (or cached in an mcache)

	// nmalloc is the cumulative count of objects allocated from
	// this mcentral, assuming all spans in mcaches are
	// fully-allocated. Written atomically, read under STW.
	nmalloc uint64
}
~~~

## 结构图

接下来，我们结合一下宏观的图示来理解一下上面的结构体之间的关联，同时对于后面的内存分配有一个简单的了解，等到后面全部讲完后，在回过头来看看这幅图，可能会对Go的内存分配有更清晰的认知

![](http://note-1253518569.cossh.myqcloud.com/20190715112554.png)

# 初始化

~~~go
func mallocinit() {
	// Initialize the heap.
	// 初始化 mheap
	mheap_.init()
	_g_ := getg()
  // 获取当前g所在的m的mcache，并初始化
	_g_.m.mcache = allocmcache()
	for i := 0x7f; i >= 0; i-- {
  var p uintptr
  switch {
  case GOARCH == "arm64" && GOOS == "darwin":
  	p = uintptr(i)<<40 | uintptrMask&(0x0013<<28)
  case GOARCH == "arm64":
  	p = uintptr(i)<<40 | uintptrMask&(0x0040<<32)
  case raceenabled:
    // The TSAN runtime requires the heap
    // to be in the range [0x00c000000000,
    // 0x00e000000000).
    p = uintptr(i)<<32 | uintptrMask&(0x00c0<<32)
    if p >= uintptrMask&0x00e000000000 {
      continue
    }
  default:
  	p = uintptr(i)<<40 | uintptrMask&(0x00c0<<32)
  }
  // 保存 arena相关属性
  hint := (*arenaHint)(mheap_.arenaHintAlloc.alloc())
  hint.addr = p
  hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
}
~~~

## mheap.init

~~~go
func (h *mheap) init() {
	h.treapalloc.init(unsafe.Sizeof(treapNode{}), nil, nil, &memstats.other_sys)
	h.spanalloc.init(unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)
	h.cachealloc.init(unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys)
	h.specialfinalizeralloc.init(unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys)
	h.specialprofilealloc.init(unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys)
	h.arenaHintAlloc.init(unsafe.Sizeof(arenaHint{}), nil, nil, &memstats.other_sys)

	// Don't zero mspan allocations. Background sweeping can
	// inspect a span concurrently with allocating it, so it's
	// important that the span's sweepgen survive across freeing
	// and re-allocating a span to prevent background sweeping
	// from improperly cas'ing it from 0.
	//
	// This is safe because mspan contains no heap pointers.
	h.spanalloc.zero = false

	// h->mapcache needs no init
	for i := range h.free {
		h.free[i].init()
		h.busy[i].init()
	}

	h.busylarge.init()
	for i := range h.central {
		h.central[i].mcentral.init(spanClass(i))
	}
}
~~~

### mcentral.init

初始化某个规格的mcentral

~~~go
// Initialize a single central free list.
func (c *mcentral) init(spc spanClass) {
	c.spanclass = spc
	c.nonempty.init()
	c.empty.init()
}
~~~

### allocmcache

mcache的初始化

~~~go
func allocmcache() *mcache {
	lock(&mheap_.lock)
	c := (*mcache)(mheap_.cachealloc.alloc())
	unlock(&mheap_.lock)
	for i := range c.alloc {
		c.alloc[i] = &emptymspan
	}
	c.next_sample = nextSample()
	return c
}
~~~

#### fixalloc.alloc

fixalloc是一个固定大小的分配器。主要用来分配一些对内存的包装的结构,比如:mspan,mcache..等等,虽然启动分配的实际使用内存是由其他内存分配器分配的。 主要分配思路为: 开始的时候一次性分配一大块内存，每次请求分配一小块，释放时放在list链表中，由于size是不变的，所以不会出现内存碎片。

~~~go
func (f *fixalloc) alloc() unsafe.Pointer {
	if f.size == 0 {
		print("runtime: use of FixAlloc_Alloc before FixAlloc_Init\n")
		throw("runtime: internal error")
	}
	
  // 如果list不要为空，直接拿
	if f.list != nil {
		v := unsafe.Pointer(f.list)
		f.list = f.list.next
		f.inuse += f.size
		if f.zero {
			memclrNoHeapPointers(v, f.size)
		}
		return v
	}
  // 如果块为空，则从系统分配中调用系统内存分配
	if uintptr(f.nchunk) < f.size {
		f.chunk = uintptr(persistentalloc(_FixAllocChunk, 0, f.stat))
		f.nchunk = _FixAllocChunk
	}
	// 从chunk中分配一个固定大小的size，释放的时候，会回归到list中
	v := unsafe.Pointer(f.chunk)
	if f.first != nil {
		f.first(f.arg, v)
	}
	f.chunk = f.chunk + f.size
	f.nchunk -= uint32(f.size)
	f.inuse += f.size
	return v
}
~~~

初始化的工作很简单：

1. 初始化heap，初始化free large对应规格的链表，初始化busyLarge链表
2. 初始化每个规格对应的mcentral
3. 初始化mcache，对mcache里面每个对应的规格进行初始化
4. 初始化 arenaHints，填充一组地址，后面根据真正的arena边界来进行扩增

# 分配

## newObject

~~~go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
~~~

### mallocgc

~~~go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	
	// Set mp.mallocing to keep from being preempted by GC.
	// 加锁防止被GC抢占
	mp := acquirem()
	if mp.mallocing != 0 {
		throw("malloc deadlock")
	}
	if mp.gsignal == getg() {
		throw("malloc during signal")
	}
	mp.mallocing = 1

	shouldhelpgc := false
	dataSize := size
	// 获取当前线程的mcache
	c := gomcache()
	var x unsafe.Pointer
	
	// 判断分配的对象是否 是nil或非指针类型
	noscan := typ == nil || typ.kind&kindNoPointers != 0
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			// 这里开始小对象的内存分配
			
			// 对齐，调整偏移量
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}
			// 如果当前mcache上绑定的tiny 块内存空间足够，直接分配，并返回
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
			// 当前mcache上的 tiny 块内存空间不足，重新分配一块 tiny 块内存
			span := c.alloc[tinySpanClass]
			
			// 尝试从 allocCache 获取内存，获取不到返回0
			v := nextFreeFast(span)
			if v == 0 {
				// 没有从 allocCache 获取到内存，netxtFree函数 尝试从 mcentral获取一个新的对应规格的快内存，替换原先内存空间不足的内存块，并分配内存，后面解析 nextFree 函数
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
			if size < c.tinyoffset || c.tiny == 0 {
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
			size = maxTinySize
		} else {
			// 这里开始 正常对象的 内存分配
			
			// 首先查表，以确定 sizeclass
			var sizeclass uint8
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
			// 找到对应 sizeclass(后面 `规格` 来代替)的span
			span := c.alloc[spc]
			// 同小对象分配一样，尝试从 allocCache 获取内存，获取不到返回0
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
		// 这里开始大对象的分配
		// 大对象的分配与 小对象 和普通对象 的分配有点不一样，大对象直接从 mheap 上分配
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			s = largeAlloc(size, needzero, noscan)
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}
	
	// bitmap标记...
	// 检查出发条件，启动垃圾回收 ...

	return x
}
~~~

整理一下 这段代码的基本思路：

1. 首先判定 对象是 大对象 还是 普通对象还是 小对象

2. 如果是 小对象

   1. 从 mcache 的alloc 找到对应 classsize 的 mspan
   2. 如果当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）
   
   3. 如果当前mspan没有足够的空间，从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后 分配并修改mspan的相关属性
   
3. 如果是普通对象，逻辑大致同小对象的 内存分配
  
   1. 首先查表，以确定 需要分配内存的对象的 sizeclass，并找到 对应 classsize的 mspan
   
   2. 如果当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）
   
   3. 如果当前mspan没有足够的空间，从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后 分配并修改mspan的相关属性
   
4. 如果是大对象，直接从mheap进行分配，这里的实现依靠 `largeAlloc` 函数实现，我们先跟一下这个函数
  
## largeAlloc

~~~go
func largeAlloc(size uintptr, needzero bool, noscan bool) *mspan {
	// print("largeAlloc size=", size, "\n")
	
  // 内存溢出判断
	if size+_PageSize < size {
		throw("out of memory")
	}
  
  // 计算出对象所需的页数
	npages := size >> _PageShift
	if size&_PageMask != 0 {
		npages++
	}

	// Deduct credit for this span allocation and sweep if
	// necessary. mHeap_Alloc will also sweep npages, so this only
	// pays the debt down to npage pages.
	deductSweepCredit(npages*_PageSize, npages)
	
  // 分配函数的具体实现
	s := mheap_.alloc(npages, makeSpanClass(0, noscan), true, needzero)
	if s == nil {
		throw("out of memory")
	}
	s.limit = s.base() + size
  // bitmap 记录分配的span
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
~~~

### mheap.alloc

~~~go
func (h *mheap) alloc(npage uintptr, spanclass spanClass, large bool, needzero bool) *mspan {
	// Don't do any operations that lock the heap on the G stack.
	// It might trigger stack growth, and the stack growth code needs
	// to be able to allocate heap.
	var s *mspan
	systemstack(func() {
		s = h.alloc_m(npage, spanclass, large)
	})

	if s != nil {
		if needzero && s.needzero != 0 {
			memclrNoHeapPointers(unsafe.Pointer(s.base()), s.npages<<_PageShift)
		}
		s.needzero = 0
	}
	return s
}
~~~

#### mheap.alloc_m

根据页数从 heap 上面分配一个新的span，并且在 HeapMap 和 HeapMapCache 上记录对象的sizeclass

~~~go
func (h *mheap) alloc_m(npage uintptr, spanclass spanClass, large bool) *mspan {
	_g_ := getg()
	if _g_ != _g_.m.g0 {
		throw("_mheap_alloc not on g0 stack")
	}
	lock(&h.lock)

	// 清理垃圾，内存块状态标记 省略...
	
	// 从 heap中获取指定页数的span
	s := h.allocSpanLocked(npage, &memstats.heap_inuse)
	if s != nil {
		// Record span info, because gc needs to be
		// able to map interior pointer to containing span.
		atomic.Store(&s.sweepgen, h.sweepgen)
		h.sweepSpans[h.sweepgen/2%2].push(s) // Add to swept in-use list.// 忽略
		s.state = _MSpanInUse
		s.allocCount = 0
		s.spanclass = spanclass
    // 重置span的状态
		if sizeclass := spanclass.sizeclass(); sizeclass == 0 {
			s.elemsize = s.npages << _PageShift
			s.divShift = 0
			s.divMul = 0
			s.divShift2 = 0
			s.baseMask = 0
		} else {
			s.elemsize = uintptr(class_to_size[sizeclass])
			m := &class_to_divmagic[sizeclass]
			s.divShift = m.shift
			s.divMul = m.mul
			s.divShift2 = m.shift2
			s.baseMask = m.baseMask
		}

		// update stats, sweep lists
		h.pagesInUse += uint64(npage)
		if large {
      // 更新 mheap中大对象的相关属性
			memstats.heap_objects++
			mheap_.largealloc += uint64(s.elemsize)
			mheap_.nlargealloc++
			atomic.Xadd64(&memstats.heap_live, int64(npage<<_PageShift))
			// Swept spans are at the end of lists.
      // 根据页数判断是busy还是 busylarge链表，并追加到末尾
			if s.npages < uintptr(len(h.busy)) {
				h.busy[s.npages].insertBack(s)
			} else {
				h.busylarge.insertBack(s)
			}
		}
	}
	// gc trace 标记，省略...
	unlock(&h.lock)
	return s
}
~~~

##### mheap.allocSpanLocked

分配一个给定大小的span，并将分配的span从freelist中移除

~~~go
func (h *mheap) allocSpanLocked(npage uintptr, stat *uint64) *mspan {
	var list *mSpanList
	var s *mspan

	// Try in fixed-size lists up to max.
  // 先尝试获取指定页数的span，如果没有，则试试页数更多的
	for i := int(npage); i < len(h.free); i++ {
		list = &h.free[i]
		if !list.isEmpty() {
			s = list.first
			list.remove(s)
			goto HaveSpan
		}
	}
	// Best fit in list of large spans.
  // 从 freelarge 上找到一个合适的span节点返回 ，下面继续分析这个函数
	s = h.allocLarge(npage) // allocLarge removed s from h.freelarge for us
	if s == nil {
    // 如果 freelarge上找不到合适的span节点，就只有从 系统 重新分配了
    // 后面继续分析这个函数
		if !h.grow(npage) {
			return nil
		}
    // 从系统分配后，再次到freelarge 上寻找合适的节点
		s = h.allocLarge(npage)
		if s == nil {
			return nil
		}
	}

HaveSpan:
  // 从 free 上面获取到了 合适页数的span
	// Mark span in use. 省略....
	
	if s.npages > npage {
		// Trim extra and put it back in the heap.
    // 创建一个 s.napges - npage 大小的span，并放回 heap
		t := (*mspan)(h.spanalloc.alloc())
		t.init(s.base()+npage<<_PageShift, s.npages-npage)
    // 更新获取到的span s 的属性
		s.npages = npage
		h.setSpan(t.base()-1, s)
		h.setSpan(t.base(), t)
		h.setSpan(t.base()+t.npages*pageSize-1, t)
		t.needzero = s.needzero
		s.state = _MSpanManual // prevent coalescing with s
		t.state = _MSpanManual
		h.freeSpanLocked(t, false, false, s.unusedsince)
		s.state = _MSpanFree
	}
	s.unusedsince = 0
	// 将s放到spans 和 arenas 数组里面
	h.setSpans(s.base(), npage, s)

	*stat += uint64(npage << _PageShift)
	memstats.heap_idle -= uint64(npage << _PageShift)

	//println("spanalloc", hex(s.start<<_PageShift))
	if s.inList() {
		throw("still in list")
	}
	return s
}
~~~

###### mheap.allocLarge

从 mheap 的 freeLarge 树上面找到一个指定page数量的span，并将该span从树上移除，找不到则返回nil

~~~go
func (h *mheap) allocLarge(npage uintptr) *mspan {
	// Search treap for smallest span with >= npage pages.
	return h.freelarge.remove(npage)
}

// 上面的 h.freelarge.remove 即调用这个函数
// 典型的二叉树寻找算法
func (root *mTreap) remove(npages uintptr) *mspan {
	t := root.treap
	for t != nil {
		if t.spanKey == nil {
			throw("treap node with nil spanKey found")
		}
		if t.npagesKey < npages {
			t = t.right
		} else if t.left != nil && t.left.npagesKey >= npages {
			t = t.left
		} else {
			result := t.spanKey
			root.removeNode(t)
			return result
		}
	}
	return nil
}
~~~

注： 在看 《Go语言学习笔记》的时候，这里的查找算法还是 对链表的 遍历查找

###### mheap.grow

在 mheap.allocSpanLocked 这个函数中，如果 freelarge上找不到合适的span节点，就只有从 系统 重新分配了，那我们接下来就继续分析一下这个函数的实现

~~~go
func (h *mheap) grow(npage uintptr) bool {
	ask := npage << _PageShift
  // 向系统申请内存，后面继续追踪 sysAlloc 这个函数
	v, size := h.sysAlloc(ask)
	if v == nil {
		print("runtime: out of memory: cannot allocate ", ask, "-byte block (", memstats.heap_sys, " in use)\n")
		return false
	}

	// Create a fake "in use" span and free it, so that the
	// right coalescing happens.
  // 创建 span 来管理刚刚申请的内存
	s := (*mspan)(h.spanalloc.alloc())
	s.init(uintptr(v), size/pageSize)
	h.setSpans(s.base(), s.npages, s)
	atomic.Store(&s.sweepgen, h.sweepgen)
	s.state = _MSpanInUse
	h.pagesInUse += uint64(s.npages)
  // 将刚刚申请的span放到 arenas 和 spans 数组里面
	h.freeSpanLocked(s, false, true, 0)
	return true
}
~~~

###### mheao.sysAlloc

 ~~~~go
func (h *mheap) sysAlloc(n uintptr) (v unsafe.Pointer, size uintptr) {
	n = round(n, heapArenaBytes)

	// First, try the arena pre-reservation.
  // 从 arena 中 获取对应大小的内存， 获取不到返回nil
	v = h.arena.alloc(n, heapArenaBytes, &memstats.heap_sys)
	if v != nil {
    // 从arena获取到需要的内存，跳转到 mapped操作
		size = n
		goto mapped
	}

	// Try to grow the heap at a hint address.
  // 尝试 从 arenaHint向下扩展内存
	for h.arenaHints != nil {
		hint := h.arenaHints
		p := hint.addr
		if hint.down {
			p -= n
		}
		if p+n < p {
			// We can't use this, so don't ask.
      // 表名 hint.down = false 不能向下扩展内存
			v = nil
		} else if arenaIndex(p+n-1) >= 1<<arenaBits {
      // 超出 heap 可寻址的内存地址，不能使用
			// Outside addressable heap. Can't use.
			v = nil
		} else {
      // 当前hint可以向下扩展内存，利用mmap向系统申请内存
			v = sysReserve(unsafe.Pointer(p), n)
		}
		if p == uintptr(v) {
			// Success. Update the hint.
			if !hint.down {
				p += n
			}
			hint.addr = p
			size = n
			break
		}
		// Failed. Discard this hint and try the next.
		//
		// TODO: This would be cleaner if sysReserve could be
		// told to only return the requested address. In
		// particular, this is already how Windows behaves, so
		// it would simply things there.
		if v != nil {
			sysFree(v, n, nil)
		}
		h.arenaHints = hint.next
		h.arenaHintAlloc.free(unsafe.Pointer(hint))
	}

	if size == 0 {
		if raceenabled {
			// The race detector assumes the heap lives in
			// [0x00c000000000, 0x00e000000000), but we
			// just ran out of hints in this region. Give
			// a nice failure.
			throw("too many address space collisions for -race mode")
		}

		// All of the hints failed, so we'll take any
		// (sufficiently aligned) address the kernel will give
		// us.
		v, size = sysReserveAligned(nil, n, heapArenaBytes)
		if v == nil {
			return nil, 0
		}

		// Create new hints for extending this region.
		hint := (*arenaHint)(h.arenaHintAlloc.alloc())
		hint.addr, hint.down = uintptr(v), true
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
		hint = (*arenaHint)(h.arenaHintAlloc.alloc())
		hint.addr = uintptr(v) + size
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
	}

	// Check for bad pointers or pointers we can't use.
	{
		var bad string
		p := uintptr(v)
		if p+size < p {
			bad = "region exceeds uintptr range"
		} else if arenaIndex(p) >= 1<<arenaBits {
			bad = "base outside usable address space"
		} else if arenaIndex(p+size-1) >= 1<<arenaBits {
			bad = "end outside usable address space"
		}
		if bad != "" {
			// This should be impossible on most architectures,
			// but it would be really confusing to debug.
			print("runtime: memory allocated by OS [", hex(p), ", ", hex(p+size), ") not in usable address space: ", bad, "\n")
			throw("memory reservation exceeds address space limit")
		}
	}

	if uintptr(v)&(heapArenaBytes-1) != 0 {
		throw("misrounded allocation in sysAlloc")
	}

	// Back the reservation.
	sysMap(v, size, &memstats.heap_sys)

mapped:
	// Create arena metadata.
  // 根据 v 的address，计算出 arenas 的L1 L2
	for ri := arenaIndex(uintptr(v)); ri <= arenaIndex(uintptr(v)+size-1); ri++ {
		l2 := h.arenas[ri.l1()]
		if l2 == nil {
      // 如果 L2 为 nil，则分配 arenas[L1]
			// Allocate an L2 arena map.
			l2 = (*[1 << arenaL2Bits]*heapArena)(persistentalloc(unsafe.Sizeof(*l2), sys.PtrSize, nil))
			if l2 == nil {
				throw("out of memory allocating heap arena map")
			}
			atomic.StorepNoWB(unsafe.Pointer(&h.arenas[ri.l1()]), unsafe.Pointer(l2))
		}
		
    // 如果 arenas[ri.L1()][ri.L2()] 不为空 说明已经实例化过了
		if l2[ri.l2()] != nil {
			throw("arena already initialized")
		}
		var r *heapArena
    // 从 arena 上分配内存
		r = (*heapArena)(h.heapArenaAlloc.alloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gc_sys))
		if r == nil {
			r = (*heapArena)(persistentalloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gc_sys))
			if r == nil {
				throw("out of memory allocating heap arena metadata")
			}
		}

		// Store atomically just in case an object from the
		// new heap arena becomes visible before the heap lock
		// is released (which shouldn't happen, but there's
		// little downside to this).
		atomic.StorepNoWB(unsafe.Pointer(&l2[ri.l2()]), unsafe.Pointer(r))
	}
	// 省略部分代码...
	return
}

 ~~~~

至此，大对象的分配流程至此结束，我们继续看一下，小对象和普通话对象的分配流程

## 小对象和普通对象分配

下面一段是 小对象和普通对象的内存查找和分配的主要函数，在上面的时候已经分析过了，下面我们就着重分析这两个函数

~~~~go
			span := c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, _, shouldhelpgc = c.nextFree(spc)
			}
~~~~

### nextFreeFast

这个函数返回 span 上可用的地址，如果找不到 则返回0

~~~go
func nextFreeFast(s *mspan) gclinkptr {
  // 计算s.allocCache从低位起有多少个0
	theBit := sys.Ctz64(s.allocCache) // Is there a free object in the allocCache?
	if theBit < 64 {
    
		result := s.freeindex + uintptr(theBit)
		if result < s.nelems {
			freeidx := result + 1
			if freeidx%64 == 0 && freeidx != s.nelems {
				return 0
			}
      // 更新bitmap、可用的 slot索引
			s.allocCache >>= uint(theBit + 1)
			s.freeindex = freeidx
			s.allocCount++
      // 返回 找到的内存的地址
			return gclinkptr(result*s.elemsize + s.base())
		}
	}
	return 0
}
~~~

### mcache.nextFree

如果 nextFreeFast 找不到 合适的内存，就会进入这个函数

nextFree 如果在cached span 里面找到未使用的object，则返回，否则，调用refill 函数，从 central 中获取对应classsize的span，然后 从新的span里面找到未使用的object返回

~~~go
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
	// 先找到 mcache 中 对应 规格的 span
  s = c.alloc[spc]
	shouldhelpgc = false
  // 在 当前span中找到合适的 index索引
	freeIndex := s.nextFreeIndex()
	if freeIndex == s.nelems {
		// The span is full.
    // freeIndex == nelems 时，表示当前span已满
		if uintptr(s.allocCount) != s.nelems {
			println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
			throw("s.allocCount != s.nelems && freeIndex == s.nelems")
		}
    // 调用refill函数，从 mcentral 中获取可用的span，并替换掉当前 mcache里面的span
		systemstack(func() {
			c.refill(spc)
		})
		shouldhelpgc = true
		s = c.alloc[spc]
		
    // 再次到新的span里面查找合适的index
		freeIndex = s.nextFreeIndex()
	}

	if freeIndex >= s.nelems {
		throw("freeIndex is not valid")
	}
	
  // 计算出来 内存地址，并更新span的属性
	v = gclinkptr(freeIndex*s.elemsize + s.base())
	s.allocCount++
	if uintptr(s.allocCount) > s.nelems {
		println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
		throw("s.allocCount > s.nelems")
	}
	return
}
~~~

#### mcache.refill

Refill 根据指定的sizeclass获取对应的span，并作为 mcache的新的sizeclass对应的span

~~~go
func (c *mcache) refill(spc spanClass) {
	_g_ := getg()

	_g_.m.locks++
	// Return the current cached span to the central lists.
	s := c.alloc[spc]

	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	
  // 判断s是不是 空的span
	if s != &emptymspan {
		s.incache = false
	}
	// 尝试从 mcentral 获取一个新的span来代替老的span
	// Get a new cached span from the central lists.
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}
	// 更新mcache的span
	c.alloc[spc] = s
	_g_.m.locks--
}
~~~

##### mcentral.cacheSpan

~~~~go
func (c *mcentral) cacheSpan() *mspan {
	// Deduct credit for this span allocation and sweep if necessary.
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	// 清理垃圾...
	lock(&c.lock)

	sg := mheap_.sweepgen
retry:
	var s *mspan
	for s = c.nonempty.first; s != nil; s = s.next {
    // if sweepgen == h->sweepgen - 2, the span needs sweeping
    // if sweepgen == h->sweepgen - 1, the span is currently being swept
    // if sweepgen == h->sweepgen, the span is swept and ready to use
    // h->sweepgen is incremented by 2 after every GC
    // 需要清理的span
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			c.nonempty.remove(s)
			c.empty.insertBack(s)
			unlock(&c.lock)
			s.sweep(true)
			goto havespan
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// we have a nonempty span that does not require sweeping, allocate from it
    // 找到片 没有被 清理的span，分配，跳转到 havespan标签继续处理
		c.nonempty.remove(s)
		c.empty.insertBack(s)
		unlock(&c.lock)
		goto havespan
	}
	
  // 对于 上一轮循环中，可能 正在清扫的span，清扫后的span可能会有有用的span，所以在这里 在进行一次遍历检查
	for s = c.empty.first; s != nil; s = s.next {
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// we have an empty span that requires sweeping,
			// sweep it and see if we can free some space in it
			c.empty.remove(s)
			// swept spans are at the end of the list
			c.empty.insertBack(s)
			unlock(&c.lock)
			s.sweep(true)
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			lock(&c.lock)
			// the span is still empty after sweep
			// it is already in the empty list, so just retry
			goto retry
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// already swept empty span,
		// all subsequent ones must also be either swept or in process of sweeping
		break
	}

	unlock(&c.lock)

	// Replenish central list if empty.
  // 找不到 合适的span，补充对应classsize的span，grow函数会调用 mheap.alloc 来填充span，上面已经分析过了，不再赘述
	s = c.grow()
	if s == nil {
		return nil
	}
	lock(&c.lock)
  // 插入到empty span list后面
	c.empty.insertBack(s)
	unlock(&c.lock)

	// At this point s is a non-empty span, queued at the end of the empty list,
	// c is unlocked.
havespan:

	cap := int32((s.npages << _PageShift) / s.elemsize)
	n := cap - int32(s.allocCount)
	if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
		throw("span has no free objects")
	}
	// Assume all objects from this span will be allocated in the
	// mcache. If it gets uncached, we'll adjust this.
	atomic.Xadd64(&c.nmalloc, int64(n))
	usedBytes := uintptr(s.allocCount) * s.elemsize
	atomic.Xadd64(&memstats.heap_live, int64(spanBytes)-int64(usedBytes))
	// 表示 span 为正在使用
	s.incache = true
	freeByteBase := s.freeindex &^ (64 - 1)
	whichByte := freeByteBase / 8
  // 更新 bitmap
	// Init alloc bits cache.
	s.refillAllocCache(whichByte)

	// Adjust the allocCache so that s.freeindex corresponds to the low bit in
	// s.allocCache.
	s.allocCache >>= s.freeindex % 64

	return s
}
~~~~

到这里，如果 从 mcentral 找不到对应的span，就开始了内存扩张之旅了，也就是我们上面分析的 `mheap.alloc`，后面的分析就同上了

## 分配小结

综上，可以看出Go的内存分配的大致流程如下

1. 首先判定 对象是 大对象 还是 普通对象还是 小对象
2. 如果是 小对象
   1. 从 mcache 的alloc 找到对应 classsize 的 mspan
   2. 如果当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）
   3. 如果当前mspan没有足够的空间，从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后 分配并修改mspan的相关属性
   4. 如果mcentral没有足够的对应的classsize的span，则去向mheap申请
   5. 如果 对应classsize的span没有了，则找一个相近的classsize的span，切割并分配
   6. 如果 找不到相近的classsize的span，则去向系统申请，并补充到mheap中
3. 如果是普通对象，逻辑大致同小对象的 内存分配
   1. 首先查表，以确定 需要分配内存的对象的 sizeclass，并找到 对应 classsize的 mspan
   2. 如果当前mspan有足够的空间，分配并修改mspan的相关属性（nextFreeFast函数中实现）
   3. 如果当前mspan没有足够的空间，从 mcentral重新获取一块 对应 classsize的 mspan，替换原先的mspan，然后 分配并修改mspan的相关属性
   4. 如果mcentral没有足够的对应的classsize的span，则去向mheap申请
   5. 如果 对应classsize的span没有了，则找一个相近的classsize的span，切割并分配
   6. 如果 找不到相近的classsize的span，则去向系统申请，并补充到mheap中
4. 如果是大对象，直接从mheap进行分配
   1. 如果 对应classsize的span没有了，则找一个相近的classsize的span，切割并分配
   2. 如果 找不到相近的classsize的span，则去向系统申请，并补充到mheap中

# 参考资料

《Go语言学习笔记》

[《图解Go语言内存分配》](https://studygolang.com/articles/20604)

[《探索Go内存管理(分配)》](https://www.jianshu.com/p/47691d870756)

[《Golang 内存管理》](http://legendtkl.com/2017/04/02/golang-alloc/)