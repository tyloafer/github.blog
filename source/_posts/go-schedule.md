---
title: 深入理解Go-goroutine的实现及调度器分析
tags:
  - Go
categories:
  - Go
date: '2019-08-31 20:04'
abbrlink: 14010
---

在学习Go的过程中，最让人惊叹的莫过于goroutine了。但是goroutine是什么，我们用`go`关键字就可以创建一个goroutine，这么多的goroutine之间，是如何调度的呢？

<!-- more -->

# 结构概览

在看Go源码的过程中，遍地可见g、p、m，我们首先就看一下这些关键字的结构及相互之间的关系

## 数据结构

这里我们仅列出来了结构体里面比较关键的一些成员

### G(gouroutine)

goroutine是运行时的最小执行单元

~~~go
type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
  // 当前g使用的栈空间，stack结构包括 [lo, hi]两个成员
	stack       stack   // offset known to runtime/cgo
  // 用于检测是否需要进行栈扩张，go代码使用
	stackguard0 uintptr // offset known to liblink
  // 用于检测是否需要进行栈扩展，原生代码使用的
	stackguard1 uintptr // offset known to liblink
  // 当前g所绑定的m
	m              *m      // current m; offset known to arm liblink
  // 当前g的调度数据，当goroutine切换时，保存当前g的上下文，用于恢复
	sched          gobuf
	// g当前的状态
	atomicstatus   uint32
  // 当前g的id
	goid           int64
  // 下一个g的地址，通过guintptr结构体的ptr set函数可以设置和获取下一个g，通过这个字段和sched.gfreeStack sched.gfreeNoStack 可以把 free g串成一个链表
	schedlink      guintptr
  // 判断g是否允许被抢占
	preempt        bool       // preemption signal, duplicates stackguard0 = stackpreempt
	// g是否要求要回到这个M执行, 有的时候g中断了恢复会要求使用原来的M执行
	lockedm        muintptr
}
~~~

### P(process)

P是M运行G所需的资源

```go
type p struct {
   lock mutex

   id          int32
   // p的状态，稍后介绍
   status      uint32 // one of pidle/prunning/...
   // 下一个p的地址，可参考 g.schedlink
   link        puintptr
   // p所关联的m
   m           muintptr   // back-link to associated m (nil if idle)
   // 内存分配的时候用的，p所属的m的mcache用的也是这个
   mcache      *mcache
  
   // Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
   // 从sched中获取并缓存的id，避免每次分配goid都从sched分配
	 goidcache    uint64
	 goidcacheend uint64

   // Queue of runnable goroutines. Accessed without lock.
   // p 本地的runnbale的goroutine形成的队列
   runqhead uint32
   runqtail uint32
   runq     [256]guintptr
   // runnext, if non-nil, is a runnable G that was ready'd by
   // the current G and should be run next instead of what's in
   // runq if there's time remaining in the running G's time
   // slice. It will inherit the time left in the current time
   // slice. If a set of goroutines is locked in a
   // communicate-and-wait pattern, this schedules that set as a
   // unit and eliminates the (potentially large) scheduling
   // latency that otherwise arises from adding the ready'd
   // goroutines to the end of the run queue.
   // 下一个执行的g，如果是nil，则从队列中获取下一个执行的g
   runnext guintptr

   // Available G's (status == Gdead)
   // 状态为 Gdead的g的列表，可以进行复用
   gfree    *g
   gfreecnt int32
}
```

### M(machine)

```go
type m struct {
   // g0是用于调度和执行系统调用的特殊g
   g0      *g     // goroutine with scheduling stack
	 // m当前运行的g
   curg          *g       // current running goroutine
   // 当前拥有的p
   p             puintptr // attached p for executing go code (nil if not executing go code)
   // 线程的 local storage
   tls           [6]uintptr   // thread-local storage
   // 唤醒m时，m会拥有这个p
   nextp         puintptr
   id            int64
   // 如果 !="", 继续运行curg
   preemptoff    string // if != "", keep curg running on this m
   // 自旋状态，用于判断m是否工作已结束，并寻找g进行工作
   spinning      bool // m is out of work and is actively looking for work
   // 用于判断m是否进行休眠状态
   blocked       bool // m is blocked on a note
	 // m休眠和唤醒通过这个，note里面有一个成员key，对这个key所指向的地址进行值的修改，进而达到唤醒和休眠的目的
   park          note
   // 所有m组成的一个链表
   alllink       *m // on allm
   // 下一个m，通过这个字段和sched.midle 可以串成一个m的空闲链表
   schedlink     muintptr
   // mcache，m拥有p的时候，会把自己的mcache给p
   mcache        *mcache
   // lockedm的对应值
   lockedg       guintptr
   // 待释放的m的list，通过sched.freem 串成一个链表
   freelink      *m      // on sched.freem
}
```

### sched

```go
type schedt struct {
   // 全局的go id分配
   goidgen  uint64
   // 记录的最后一次从i/o中查询g的时间
   lastpoll uint64

   lock mutex

   // When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
   // sure to call checkdead().
	 // m的空闲链表，结合m.schedlink 就可以组成一个空闲链表了
   midle        muintptr // idle m's waiting for work
   nmidle       int32    // number of idle m's waiting for work
   nmidlelocked int32    // number of locked m's waiting for work
   // 下一个m的id，也用来记录创建的m数量
   mnext        int64    // number of m's that have been created and next M ID
   // 最多允许的m的数量
   maxmcount    int32    // maximum number of m's allowed (or die)
   nmsys        int32    // number of system m's not counted for deadlock
   // free掉的m的数量，exit的m的数量
   nmfreed      int64    // cumulative number of freed m's

   ngsys uint32 // number of system goroutines; updated atomically

   pidle      puintptr // idle p's
   npidle     uint32
   nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

   // Global runnable queue.
   // 这个就是全局的g的队列了，如果p的本地队列没有g或者太多，会跟全局队列进行平衡
   // 根据runqhead可以获取队列头的g，然后根据g.schedlink 获取下一个，从而形成了一个链表
   runqhead guintptr
   runqtail guintptr
   runqsize int32

   // freem is the list of m's waiting to be freed when their
   // m.exited is set. Linked through m.freelink.
   // 等待释放的m的列表
   freem *m
}
```

在这里插一下状态的解析

### g.status

- _Gidle: goroutine刚刚创建还没有初始化
- _Grunnable: goroutine处于运行队列中，但是还没有运行，没有自己的栈
- _Grunning:  这个状态的g可能处于运行用户代码的过程中，拥有自己的m和p
- _Gsyscall: 运行systemcall中
- _Gwaiting: 这个状态的goroutine正在阻塞中，类似于等待channel
- _Gdead: 这个状态的g没有被使用，有可能是刚刚退出，也有可能是正在初始化中
- _Gcopystack: 表示g当前的栈正在被移除，新栈分配中

### p.status

- _Pidle: 空闲状态，此时p不绑定m
- _Prunning: m获取到p的时候，p的状态就是这个状态了，然后m可以使用这个p的资源运行g
- _Psyscall: 当go调用原生代码，原生代码又反过来调用go的时候，使用的p就会变成此态
- _Pdead: 当运行中，需要减少p的数量时，被减掉的p的状态就是这个了

### m.status

m的status没有p、g的那么明确，但是在运行流程的分析中，主要有以下几个状态

- 运行中: 拿到p，执行g的过程中
- 运行原生代码: 正在执行原声代码或者阻塞的syscall
- 休眠中: m发现无待运行的g时，进入休眠，并加入到空闲列表中
- 自旋中(spining): 当前工作结束，正在寻找下一个待运行的g

在上面的结构中，存在很多的链表，g m p结构中还有指向对方地址的成员，那么他们的关系到底是什么样的

![](http://note-1253518569.cossh.myqcloud.com/20190830155713.png)

我们可以从上图，简单的表述一下 m p g的关系

# 流程概览

从下图，可以简单的一窥go的整个调度流程的大概

![](http://note-1253518569.cossh.myqcloud.com/20190830154901.png)

接下来我们就从源码的角度来具体的分析整个调度流程（本人汇编不照，汇编方面的就不分析了🤪）

# 源码分析

## 初始化

go的启动流程分为4步

1. call osinit， 这里就是设置了全局变量ncpu = cpu核心数量
2. call schedinit
3. make & queue new G （runtime.newproc, go func()也是调用这个函数来创建goroutine）
4. call runtime·mstart

其中，schedinit 就是调度器的初始化，出去schedinit 中对内存分配，垃圾回收等操作，针对调度器的初始化大致就是初始化自身，设置最大的maxmcount， 确定p的数量并初始化这些操作

### schedinit

schedinit这里对当前m进行了初始化，并根据osinit获取到的cpu核数和设置的`GOMAXPROCS` 确定p的数量，并进行初始化

```go
func schedinit() {
	// 从TLS或者专用寄存器获取当前g的指针类型
	_g_ := getg()
	// 设置m最大的数量
	sched.maxmcount = 10000

	// 初始化栈的复用空间
	stackinit()
	// 初始化当前m
	mcommoninit(_g_.m)

	// osinit的时候会设置 ncpu这个全局变量，这里就是根据cpu核心数和参数GOMAXPROCS来确定p的数量
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	// 生成设定数量的p
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```

### mcommoninit

```go
func mcommoninit(mp *m) {
	_g_ := getg()

	lock(&sched.lock)
	// 判断mnext的值是否溢出，mnext需要赋值给m.id
	if sched.mnext+1 < sched.mnext {
		throw("runtime: thread ID overflow")
	}
	mp.id = sched.mnext
	sched.mnext++
	// 判断m的数量是否比maxmcount设定的要多，如果超出直接报异常
	checkmcount()
	// 创建一个新的g用于处理signal，并分配栈
	mpreinit(mp)
	if mp.gsignal != nil {
		mp.gsignal.stackguard1 = mp.gsignal.stack.lo + _StackGuard
	}

	// Add to allm so garbage collector doesn't free g->m
	// when it is just in a register or thread-local storage.
	// 接下来的两行，首先将当前m放到allm的头，然后原子操作，将当前m的地址，赋值给m，这样就将当前m添加到了allm链表的头了
	mp.alllink = allm

	// NumCgoCall() iterates over allm w/o schedlock,
	// so we need to publish it safely.
	atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))
	unlock(&sched.lock)

	// Allocate memory to hold a cgo traceback if the cgo call crashes.
	if iscgo || GOOS == "solaris" || GOOS == "windows" {
		mp.cgoCallers = new(cgoCallers)
	}
}
```

在这里就开始涉及到了m链表了，这个链表可以如下图表示，其他的p g链表可以参考，只是使用的结构体的字段不一样

#### 3.1.1.2. allm链表示意图

![](http://note-1253518569.cossh.myqcloud.com/20190831105630.png)

#### 3.1.1.3. procresize

更改p的数量，多退少补的原则，在初始化过程中，由于最开始是没有p的，所以这里的作用就是初始化设定数量的p了

`procesize` 不仅在初始化的时候会调用，当用户手动调用 `runtime.GOMAXPROCS` 的时候，会重新设定 nprocs，然后执行 `startTheWorld()`， `startTheWorld()`会是使用新的 nprocs 再次调用`procresize` 这个方法

```go
func procresize(nprocs int32) *p {
	old := gomaxprocs
	if old < 0 || nprocs <= 0 {
		throw("procresize: invalid arg")
	}
	// update statistics
	now := nanotime()
	if sched.procresizetime != 0 {
		sched.totaltime += int64(old) * (now - sched.procresizetime)
	}
	sched.procresizetime = now

	// Grow allp if necessary.
	// 如果新给的p的数量比原先的p的数量多，则新建增长的p
	if nprocs > int32(len(allp)) {
		// Synchronize with retake, which could be running
		// concurrently since it doesn't run on a P.
		lock(&allpLock)
		// 判断allp 的cap是否满足增长后的长度，满足就直接使用，不满足，则需要扩张这个slice
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			// Copy everything up to allp's cap so we
			// never lose old allocated Ps.
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
		unlock(&allpLock)
	}

	// initialize new P's
	// 初始化新增的p
	for i := int32(0); i < nprocs; i++ {
		pp := allp[i]
		if pp == nil {
			pp = new(p)
			pp.id = i
			pp.status = _Pgcstop
			pp.sudogcache = pp.sudogbuf[:0]
			for i := range pp.deferpool {
				pp.deferpool[i] = pp.deferpoolbuf[i][:0]
			}
			pp.wbBuf.reset()
			// allp是一个slice，直接将新增的p放到对应的索引下面就ok了
			atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
		}
		if pp.mcache == nil {
			// 初始化时，old=0，第一个新建的p给当前的m使用
			if old == 0 && i == 0 {
				if getg().m.mcache == nil {
					throw("missing mcache?")
				}
				pp.mcache = getg().m.mcache // bootstrap
			} else {
				// 为p分配内存
				pp.mcache = allocmcache()
			}
		}
	}

	// free unused P's
	// 释放掉多余的p，当新设置的p的数量，比原先设定的p的数量少的时候，会走到这个流程
	// 通过 runtime.GOMAXPROCS 就可以动态的修改nprocs
	for i := nprocs; i < old; i++ {
		p := allp[i]
		// move all runnable goroutines to the global queue
		// 把当前p的运行队列里的g转移到全局的g的队列
		for p.runqhead != p.runqtail {
			// pop from tail of local queue
			p.runqtail--
			gp := p.runq[p.runqtail%uint32(len(p.runq))].ptr()
			// push onto head of global queue
			globrunqputhead(gp)
		}
		// 把runnext里的g也转移到全局队列
		if p.runnext != 0 {
			globrunqputhead(p.runnext.ptr())
			p.runnext = 0
		}
		// if there's a background worker, make it runnable and put
		// it on the global queue so it can clean itself up
		// 如果有gc worker的话，修改g的状态，然后再把它放到全局队列中
		if gp := p.gcBgMarkWorker.ptr(); gp != nil {
			casgstatus(gp, _Gwaiting, _Grunnable)
			globrunqput(gp)
			// This assignment doesn't race because the
			// world is stopped.
			p.gcBgMarkWorker.set(nil)
		}
		// sudoig的buf和cache，以及deferpool全部清空
		for i := range p.sudogbuf {
			p.sudogbuf[i] = nil
		}
		p.sudogcache = p.sudogbuf[:0]
		for i := range p.deferpool {
			for j := range p.deferpoolbuf[i] {
				p.deferpoolbuf[i][j] = nil
			}
			p.deferpool[i] = p.deferpoolbuf[i][:0]
		}
		// 释放掉当前p的mcache
		freemcache(p.mcache)
		p.mcache = nil
		// 把当前p的gfree转移到全局
		gfpurge(p)
		// 修改p的状态，让他自生自灭去了
		p.status = _Pdead
		// can't free P itself because it can be referenced by an M in syscall
	}

	// Trim allp.
	if int32(len(allp)) != nprocs {
		lock(&allpLock)
		allp = allp[:nprocs]
		unlock(&allpLock)
	}
	// 判断当前g是否有p，有的话更改当前使用的p的状态，继续使用
	_g_ := getg()
	if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
		// continue to use the current P
		_g_.m.p.ptr().status = _Prunning
	} else {
		// release the current P and acquire allp[0]
		// 如果当前g有p，但是拥有的是已经释放的p，则不再使用这个p，重新分配
		if _g_.m.p != 0 {
			_g_.m.p.ptr().m = 0
		}
		// 分配allp[0]给当前g使用
		_g_.m.p = 0
		_g_.m.mcache = nil
		p := allp[0]
		p.m = 0
		p.status = _Pidle
		// 将p m g绑定，并把m.mcache指向p.mcache，并修改p的状态为_Prunning
		acquirep(p)
	}
	var runnablePs *p
	for i := nprocs - 1; i >= 0; i-- {
		p := allp[i]
		if _g_.m.p.ptr() == p {
			continue
		}
		p.status = _Pidle
		// 根据 runqempty 来判断当前p的g运行队列是否为空
		if runqempty(p) {
			// g运行队列为空的p，放到 sched的pidle队列里面
			pidleput(p)
		} else {
			// g 运行队列不为空的p，组成一个可运行队列，并最后返回
			p.m.set(mget())
			p.link.set(runnablePs)
			runnablePs = p
		}
	}
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
	return runnablePs
}
```

- runqempty: 这个函数比较简单，就不深究了，就是根据 p.runqtail == p.runqhead 和 p.runnext 来判断有没有待运行的g
- pidleput: 将当前的p设置为 sched.pidle，然后根据p.link将空闲p串联起来，可参考上图allm的链表示意图

## 任务

创建一个goroutine，只需要使用 `go func` 就可以了，编译器会将`go func` 翻译成 `newproc` 进行调用，那么新建的任务是如何调用的呢，我们从创建开始进行跟踪

### newproc

`newproc` 函数获取了参数和当前g的pc信息，并通过g0调用`newproc1`去真正的执行创建或获取可用的g

```go
func newproc(siz int32, fn *funcval) {
	// 获取第一参数地址
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	// 获取当前执行的g
	gp := getg()
	// 获取当前g的pc
	pc := getcallerpc()
	systemstack(func() {
		// 使用g0去执行newproc1函数
		newproc1(fn, (*uint8)(argp), siz, gp, pc)
	})
}
```

### newproc1

newporc1 的作用就是创建或者获取一个空间的g，初始化这个g，并尝试寻找一个p和m去执行g

```go
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
	_g_ := getg()

	if fn == nil {
		_g_.m.throwing = -1 // do not dump full stacks
		throw("go of nil func value")
	}
	// 加锁禁止被抢占
	_g_.m.locks++ // disable preemption because it can be holding p in a local var
	siz := narg
	siz = (siz + 7) &^ 7

	// We could allocate a larger initial stack if necessary.
	// Not worth it: this is almost always an error.
	// 4*sizeof(uintreg): extra space added below
	// sizeof(uintreg): caller's LR (arm) or return address (x86, in gostartcall).
	// 如果参数过多，则直接抛出异常，栈大小是2k
	if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
		throw("newproc: function arguments too large for new goroutine")
	}

	_p_ := _g_.m.p.ptr()
	// 尝试获取一个空闲的g，如果获取不到，则新建一个，并添加到allg里面
	// gfget首先会尝试从p本地获取空闲的g，如果本地没有的话，则从全局获取一堆平衡到本地p
	newg := gfget(_p_)
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		// 新建的g，添加到全局的 allg里面，allg是一个slice， append进去即可
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	// 判断获取的g的栈是否正常
	if newg.stack.hi == 0 {
		throw("newproc1: newg missing stack")
	}
	// 判断g的状态是否正常
	if readgstatus(newg) != _Gdead {
		throw("newproc1: new g is not Gdead")
	}
	// 预留一点空间，防止读取超出一点点
	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
	// 空间大小进行对齐
	totalSize += -totalSize & (sys.SpAlign - 1) // align to spAlign
	sp := newg.stack.hi - totalSize
	spArg := sp
	// usesLr 为0，这里不执行
	if usesLR {
		// caller's LR
		*(*uintptr)(unsafe.Pointer(sp)) = 0
		prepGoExitFrame(sp)
		spArg += sys.MinFrameSize
	}
	if narg > 0 {
		// 将参数拷贝入栈
		memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))
		// ... 省略 ...
	}
	// 初始化用于保存现场的区域及初始化基本状态
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
	// 这里保存了goexit的地址，在用户函数执行完成后，会根据pc来执行goexit
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	// 这里调整 sched 信息，pc = goexit的地址
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg) {
		atomic.Xadd(&sched.ngsys, +1)
	}
	newg.gcscanvalid = false
	casgstatus(newg, _Gdead, _Grunnable)
	// 如果p缓存的goid已经用完，本地再从sched批量获取一点
	if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	// 分配goid
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	// 把新的g放到 p 的可运行g队列中
	runqput(_p_, newg, true)
	// 判断是否有空闲p，且是否需要唤醒一个m来执行g
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		wakep()
	}
	_g_.m.locks--
	if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
		_g_.stackguard0 = stackPreempt
	}
}
```

#### gfget

这个函数的逻辑比较简单，就是看一下p有没有空闲的g，没有则去全局的freeg队列查找，这里就涉及了p本地和全局平衡的一个交互了

```go
func gfget(_p_ *p) *g {
retry:
	gp := _p_.gfree
	// 本地的g队列为空，且全局队列不为空，则从全局队列一次获取至多32个下来，如果全局队列不够就算了
	if gp == nil && (sched.gfreeStack != nil || sched.gfreeNoStack != nil) {
		lock(&sched.gflock)
		for _p_.gfreecnt < 32 {
			if sched.gfreeStack != nil {
				// Prefer Gs with stacks.
				gp = sched.gfreeStack
				sched.gfreeStack = gp.schedlink.ptr()
			} else if sched.gfreeNoStack != nil {
				gp = sched.gfreeNoStack
				sched.gfreeNoStack = gp.schedlink.ptr()
			} else {
				break
			}
			_p_.gfreecnt++
			sched.ngfree--
			gp.schedlink.set(_p_.gfree)
			_p_.gfree = gp
		}
		// 已经从全局拿了g了，再去从头开始判断
		unlock(&sched.gflock)
		goto retry
	}
	// 如果拿到了g，则判断g是否有栈，没有栈就分配
	// 栈的分配跟内存分配差不多，首先创建几个固定大小的栈的数组，然后到指定大小的数组里面去分配就ok了，过大则直接全局分配
	if gp != nil {
		_p_.gfree = gp.schedlink.ptr()
		_p_.gfreecnt--
		if gp.stack.lo == 0 {
			// Stack was deallocated in gfput. Allocate a new one.
			systemstack(func() {
				gp.stack = stackalloc(_FixedStack)
			})
			gp.stackguard0 = gp.stack.lo + _StackGuard
		} else {
			// ... 省略 ...
		}
	}
	// 注意： 如果全局没有g，p也没有g，则返回的gp还是nil
	return gp
}
```

#### runqput

runqput会把g放到p的本地队列或者p.runnext，如果p的本地队列过长，则把g到全局队列，同时平衡p本地队列的一半到全局

```go
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}
	// 如果next为true，则放入到p.runnext里面，并把原先runnext的g交换出来
	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	h := atomic.Load(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	// 判断p的队列的长度是否超了， runq是一个长度为256的数组，超出的话就会放到全局队列了
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.Store(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	// 把g放到全局队列
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```

#### runqputslow

```go
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
	var batch [len(_p_.runq)/2 + 1]*g

	// First, grab a batch from local queue.
	n := t - h
	n = n / 2
	if n != uint32(len(_p_.runq)/2) {
		throw("runqputslow: queue is not full")
	}
	// 获取p后面的一半
	for i := uint32(0); i < n; i++ {
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr()
	}
	if !atomic.Cas(&_p_.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
	batch[n] = gp

	// Link the goroutines.
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}

	// Now put the batch on global queue.
	// 放到全局队列队尾
	lock(&sched.lock)
	globrunqputbatch(batch[0], batch[n], int32(n+1))
	unlock(&sched.lock)
	return true
}
```

新建任务至此基本结束，创建完成任务后，等待调度执行就好了，从上面可以看出，任务的优先级是 p.runnext > p.runq > sched.runq

g从创建到执行结束并放入free队列中的状态转换大致如下图所示

![](http://note-1253518569.cossh.myqcloud.com/20190831160849.png)

### wakep

当 newproc1创建完任务后，会尝试唤醒m来执行任务

```go
func wakep() {
	// be conservative about spinning threads
	// 一次应该只有一个m在spining，否则就退出
	if !atomic.Cas(&sched.nmspinning, 0, 1) {
		return
	}
	// 调用startm来执行
	startm(nil, true)
}
```

### startm

调度m或者创建m来运行p，如果p==nil，就会尝试获取一个空闲p，p的队列中有g，拿到p后才能拿到g

```go
func startm(_p_ *p, spinning bool) {
	lock(&sched.lock)
	if _p_ == nil {
		// 如果没有指定p, 则从sched.pidle获取空闲的p
		_p_ = pidleget()
		if _p_ == nil {
			unlock(&sched.lock)
			// 如果没有获取到p，重置nmspinning
			if spinning {
				// The caller incremented nmspinning, but there are no idle Ps,
				// so it's okay to just undo the increment and give up.
				if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
					throw("startm: negative nmspinning")
				}
			}
			return
		}
	}
	// 首先尝试从 sched.midle获取一个空闲的m
	mp := mget()
	unlock(&sched.lock)
	if mp == nil {
		// 如果获取不到空闲的m，则创建一个 mspining = true的m，并将p绑定到m上，直接返回
		var fn func()
		if spinning {
			// The caller incremented nmspinning, so set m.spinning in the new M.
			fn = mspinning
		}
		newm(fn, _p_)
		return
	}
	// 判断获取到的空闲m是否是spining状态
	if mp.spinning {
		throw("startm: m is spinning")
	}
	// 判断获取到的m是否有p
	if mp.nextp != 0 {
		throw("startm: m has p")
	}
	if spinning && !runqempty(_p_) {
		throw("startm: p has runnable gs")
	}
	// The caller incremented nmspinning, so set m.spinning in the new M.
	// 调用函数的父函数已经增加了nmspinning， 这里只需要设置m.spining就ok了，同时把p绑上来
	mp.spinning = spinning
	mp.nextp.set(_p_)
	// 唤醒m
	notewakeup(&mp.park)
}
```

#### newm

newm 通过allocm函数来创建新m

```go
func newm(fn func(), _p_ *p) {
	// 新建一个m
	mp := allocm(_p_, fn)
	// 为这个新建的m绑定指定的p
	mp.nextp.set(_p_)
	// ... 省略 ...
	// 创建系统线程
	newm1(mp)
}
```

#### new1m

```go
func newm1(mp *m) {
	// runtime cgo包会把iscgo设置为true，这里不分析
	if iscgo {
		var ts cgothreadstart
		if _cgo_thread_start == nil {
			throw("_cgo_thread_start missing")
		}
		ts.g.set(mp.g0)
		ts.tls = (*uint64)(unsafe.Pointer(&mp.tls[0]))
		ts.fn = unsafe.Pointer(funcPC(mstart))
		if msanenabled {
			msanwrite(unsafe.Pointer(&ts), unsafe.Sizeof(ts))
		}
		execLock.rlock() // Prevent process clone.
		asmcgocall(_cgo_thread_start, unsafe.Pointer(&ts))
		execLock.runlock()
		return
	}
	execLock.rlock() // Prevent process clone.
	newosproc(mp)
	execLock.runlock()
}
```

#### newosproc

newosproc 创建一个新的系统线程，并执行mstart_stub函数，之后调用`mstart`函数进入调度，后面在执行流程会分析

```go
func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
	// Initialize an attribute object.
	var attr pthreadattr
	var err int32
	err = pthread_attr_init(&attr)

	// Finally, create the thread. It starts at mstart_stub, which does some low-level
	// setup and then calls mstart.
	var oset sigset
	sigprocmask(_SIG_SETMASK, &sigset_all, &oset)
	// 创建线程，并传入启动启动函数 mstart_stub， mstart_stub 之后调用mstart
	err = pthread_create(&attr, funcPC(mstart_stub), unsafe.Pointer(mp))
	sigprocmask(_SIG_SETMASK, &oset, nil)
	if err != 0 {
		write(2, unsafe.Pointer(&failthreadcreate[0]), int32(len(failthreadcreate)))
		exit(1)
	}
}
```

#### allocm

allocm这里首先会释放 sched的freem，然后再去创建m，并初始化m

```go
func allocm(_p_ *p, fn func()) *m {
	_g_ := getg()
	_g_.m.locks++ // disable GC because it can be called from sysmon
	if _g_.m.p == 0 {
		acquirep(_p_) // temporarily borrow p for mallocs in this function
	}

	// Release the free M list. We need to do this somewhere and
	// this may free up a stack we can use.
	// 首先释放掉freem列表
	if sched.freem != nil {
		lock(&sched.lock)
		var newList *m
		for freem := sched.freem; freem != nil; {
			if freem.freeWait != 0 {
				next := freem.freelink
				freem.freelink = newList
				newList = freem
				freem = next
				continue
			}
			stackfree(freem.g0.stack)
			freem = freem.freelink
		}
		sched.freem = newList
		unlock(&sched.lock)
	}

	mp := new(m)
	// 启动函数，根据startm调用来看，这个fn就是 mspinning， 会将m.mspinning设置为true
	mp.mstartfn = fn
	// 初始化m，上面已经分析了
	mcommoninit(mp)
	// In case of cgo or Solaris or Darwin, pthread_create will make us a stack.
	// Windows and Plan 9 will layout sched stack on OS stack.
	// 为新的m创建g0
	if iscgo || GOOS == "solaris" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * sys.StackGuardMultiplier)
	}
	// 为mp的g0绑定自己
	mp.g0.m = mp
	// 如果当前的m所绑定的是参数传递过来的p，解除绑定，因为参数传递过来的p稍后要绑定新建的m
	if _p_ == _g_.m.p.ptr() {
		releasep()
	}

	_g_.m.locks--
	if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
		_g_.stackguard0 = stackPreempt
	}

	return mp
}
```

#### notewakeup

```go
func notewakeup(n *note) {
	var v uintptr
	// 设置m 为locked
	for {
		v = atomic.Loaduintptr(&n.key)
		if atomic.Casuintptr(&n.key, v, locked) {
			break
		}
	}

	// Successfully set waitm to locked.
	// What was it before?
	// 根据m的原先的状态，来判断后面的执行流程，0则直接返回，locked则冲突，否则认为是wating，唤醒
	switch {
	case v == 0:
		// Nothing was waiting. Done.
	case v == locked:
		// Two notewakeups! Not allowed.
		throw("notewakeup - double wakeup")
	default:
		// Must be the waiting m. Wake it up.
		// 唤醒系统线程
		semawakeup((*m)(unsafe.Pointer(v)))
	}
}
```

至此的话，创建完任务g后，将g放入了p的local队列或者是全局队列，然后开始获取了一个空闲的m或者新建一个m来执行g，m, p, g 都已经准备完成了，下面就是开始调度，来运行任务g了

## 执行

在startm函数分析的过程中会，可以看到，有两种获取m的方式

- 新建： 这时候执行newm1下的newosproc，同时最终调用mstart来执行调度
- 唤醒空闲m：从休眠的地方继续执行

m执行g有两个起点，一个是线程启动函数 `mstart`， 另一个则是休眠被唤醒后的调度`schedule`了，我们从头开始，也就是`mstart`， `mstart` 走到最后也是 `schedule` 调度

### mstart

```go
func mstart() {
	_g_ := getg()

	osStack := _g_.stack.lo == 0
	if osStack {
		// Initialize stack bounds from system stack.
		// Cgo may have left stack size in stack.hi.
		// minit may update the stack bounds.
		// 从系统堆栈上直接划出所需的范围
		size := _g_.stack.hi
		if size == 0 {
			size = 8192 * sys.StackGuardMultiplier
		}
		_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
		_g_.stack.lo = _g_.stack.hi - size + 1024
	}
	// Initialize stack guards so that we can start calling
	// both Go and C functions with stack growth prologues.
	_g_.stackguard0 = _g_.stack.lo + _StackGuard
	_g_.stackguard1 = _g_.stackguard0
	// 调用mstart1来处理
	mstart1()

	// Exit this thread.
	if GOOS == "windows" || GOOS == "solaris" || GOOS == "plan9" || GOOS == "darwin" {
		// Window, Solaris, Darwin and Plan 9 always system-allocate
		// the stack, but put it in _g_.stack before mstart,
		// so the logic above hasn't set osStack yet.
		osStack = true
	}
	// 退出m，正常情况下mstart1调用schedule() 时，是不再返回的，所以，不用担心系统线程的频繁创建退出
	mexit(osStack)
}
```

### mstart1

```go
func mstart1() {
	_g_ := getg()

	if _g_ != _g_.m.g0 {
		throw("bad runtime·mstart")
	}

	// Record the caller for use as the top of stack in mcall and
	// for terminating the thread.
	// We're never coming back to mstart1 after we call schedule,
	// so other calls can reuse the current frame.
	// 保存调用者的pc sp等信息
	save(getcallerpc(), getcallersp())
	asminit()
	// 初始化m的sigal的栈和mask
	minit()

	// Install signal handlers; after minit so that minit can
	// prepare the thread to be able to handle the signals.
	// 安装sigal处理器
	if _g_.m == &m0 {
		mstartm0()
	}
	// 如果设置了mstartfn，就先执行这个
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}

	if _g_.m.helpgc != 0 {
		_g_.m.helpgc = 0
		stopm()
	} else if _g_.m != &m0 {
		// 获取nextp
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
	schedule()
}
```

#### acquirep

acquirep 函数主要是改变p的状态，绑定 m p，通过吧p的mcache与m共享

```go
func acquirep(_p_ *p) {
	// Do the part that isn't allowed to have write barriers.
	acquirep1(_p_)

	// have p; write barriers now allowed
	_g_ := getg()
	// 把p的mcache与m共享
	_g_.m.mcache = _p_.mcache
}
```

#### acquirep1

```go
func acquirep1(_p_ *p) {
	_g_ := getg()

	// 让m p互相绑定
	_g_.m.p.set(_p_)
	_p_.m.set(_g_.m)
	_p_.status = _Prunning
}
```

#### schedule

开始进入到调度函数了，这是一个由schedule、execute、goroutine fn、goexit构成的逻辑循环，就算m是唤醒后，也是从设置的断点开始执行

```go
func schedule() {
	_g_ := getg()

	if _g_.m.locks != 0 {
		throw("schedule: holding locks")
	}
	// 如果有lockg，停止执行当前的m
	if _g_.m.lockedg != 0 {
		// 解除lockedm的锁定，并执行当前g
		stoplockedm()
		execute(_g_.m.lockedg.ptr(), false) // Never returns.
	}

	// We should not schedule away from a g that is executing a cgo call,
	// since the cgo call is using the m's g0 stack.
	if _g_.m.incgo {
		throw("schedule: in cgo")
	}

top:
	// gc 等待
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}

	var gp *g
	var inheritTime bool

	if gp == nil {
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		// 为了保证公平，每隔61次，从全局队列上获取g
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		// 全局队列上获取不到待运行的g，则从p local队列中获取
		gp, inheritTime = runqget(_g_.m.p.ptr())
		if gp != nil && _g_.m.spinning {
			throw("schedule: spinning with local work")
		}
	}
	if gp == nil {
		// 如果p local获取不到待运行g，则开始查找，这个函数会从 全局 io poll， p locl和其他p local获取待运行的g，后面详细分析
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	// This thread is going to run a goroutine and is not spinning anymore,
	// so if it was marked as spinning we need to reset it now and potentially
	// start a new spinning M.
	if _g_.m.spinning {
		// 如果m是自旋状态，取消自旋
		resetspinning()
	}

	if gp.lockedm != 0 {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		// 如果g有lockedm，则休眠上交p，休眠m，等待新的m，唤醒后从这里开始执行，跳转到top
		startlockedm(gp)
		goto top
	}
	// 开始执行这个g
	execute(gp, inheritTime)
}
```

##### stoplockedm

因为当前的m绑定了lockedg，而当前g不是指定的lockedg，所以这个m不能执行，上交当前m绑定的p，并且休眠m直到调度lockedg

```go
func stoplockedm() {
	_g_ := getg()

	if _g_.m.lockedg == 0 || _g_.m.lockedg.ptr().lockedm.ptr() != _g_.m {
		throw("stoplockedm: inconsistent locking")
	}
	if _g_.m.p != 0 {
		// Schedule another M to run this p.
		// 释放当前p
		_p_ := releasep()
		handoffp(_p_)
	}
	incidlelocked(1)
	// Wait until another thread schedules lockedg again.
	notesleep(&_g_.m.park)
	noteclear(&_g_.m.park)
	status := readgstatus(_g_.m.lockedg.ptr())
	if status&^_Gscan != _Grunnable {
		print("runtime:stoplockedm: g is not Grunnable or Gscanrunnable\n")
		dumpgstatus(_g_)
		throw("stoplockedm: not runnable")
	}
	// 上交了当前的p，将nextp设置为可执行的p
	acquirep(_g_.m.nextp.ptr())
	_g_.m.nextp = 0
}
```

##### startlockedm

调度 lockedm去运行lockedg

```GO
func startlockedm(gp *g) {
	_g_ := getg()

	mp := gp.lockedm.ptr()
	if mp == _g_.m {
		throw("startlockedm: locked to me")
	}
	if mp.nextp != 0 {
		throw("startlockedm: m has p")
	}
	// directly handoff current P to the locked m
	incidlelocked(-1)
	// 移交当前p给lockedm，并设置为lockedm.nextp，以便于lockedm唤醒后，可以获取
	_p_ := releasep()
	mp.nextp.set(_p_)
	// m被唤醒后，从m休眠的地方开始执行，也就是schedule()函数中
	notewakeup(&mp.park)
	stopm()
}
```

##### handoffp

```go
func handoffp(_p_ *p) {
	// handoffp must start an M in any situation where
	// findrunnable would return a G to run on _p_.

	// if it has local work, start it straight away
	if !runqempty(_p_) || sched.runqsize != 0 {
		// 调用startm开始调度
		startm(_p_, false)
		return
	}

	// no local work, check that there are no spinning/idle M's,
	// otherwise our help is not required
	// 判断有没有正在寻找p的m以及有没有空闲的p
	if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
		startm(_p_, true)
		return
	}
	lock(&sched.lock)

	if _p_.runSafePointFn != 0 && atomic.Cas(&_p_.runSafePointFn, 1, 0) {
		sched.safePointFn(_p_)
		sched.safePointWait--
		if sched.safePointWait == 0 {
			notewakeup(&sched.safePointNote)
		}
	}
	// 如果 全局待运行g队列不为空，尝试使用startm进行调度
	if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(_p_, false)
		return
	}
	// If this is the last running P and nobody is polling network,
	// need to wakeup another M to poll network.
	if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
		unlock(&sched.lock)
		startm(_p_, false)
		return
	}
	// 把p放入到全局的空闲队列，放回队列就不多说了，参考allm的放回
	pidleput(_p_)
	unlock(&sched.lock)
}
```

##### execute

开始执行g的代码了

```go
func execute(gp *g, inheritTime bool) {
	_g_ := getg()
	// 更改g的状态，并不允许抢占
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		// 调度计数
		_g_.m.p.ptr().schedtick++
	}
	_g_.m.curg = gp
	gp.m = _g_.m
	// 开始执行g的代码了
	gogo(&gp.sched)
}
```

##### gogo

gogo函数承载的作用就是切换到g的栈，开始执行g的代码，汇编内容就不分析了，但是有一个疑问就是，gogo执行完函数后，怎么再次进入调度呢？

我们回到`newproc1`函数的L63 `newg.sched.pc = funcPC(goexit) + sys.PCQuantum` ，这里保存了pc的质地为goexit的地址，所以当执行完用户代码后，就会进入 `goexit` 函数

##### goexit0

goexit 在汇编层面就是调用 `runtime.goexit1`，而goexit1通过 mcall 调用了`goexit0` 所以这里直接分析了`goexit0`

`goexit0` 重置g的状态，并重新进行调度，这样就调度就又回到了`schedule()` 了，开始循环往复的调度

```go
func goexit0(gp *g) {
	_g_ := getg()
	// 转换g的状态为dead，以放回空闲列表
	casgstatus(gp, _Grunning, _Gdead)
	if isSystemGoroutine(gp) {
		atomic.Xadd(&sched.ngsys, -1)
	}
	// 清空g的状态
	gp.m = nil
	locked := gp.lockedm != 0
	gp.lockedm = 0
	_g_.m.lockedg = 0
	gp.paniconfault = false
	gp._defer = nil // should be true already but just in case.
	gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
	gp.writebuf = nil
	gp.waitreason = 0
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	// Note that gp's stack scan is now "valid" because it has no
	// stack.
	gp.gcscanvalid = true
	dropg()

	// 把g放回空闲列表，以备复用
	gfput(_g_.m.p.ptr(), gp)
	// 再次进入调度循环
	schedule()
}
```

至此，单次调度结束，再次进入调度，循环往复

##### findrunnable

```go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// The conditions here and in handoffp must agree: if
	// findrunnable would return a G to run, handoffp must start
	// an M.

top:
	_p_ := _g_.m.p.ptr()

	// local runq
	// 从p local 去获取g
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// global runq
	// 从全局的待运行d队列获取
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
	// 看看netpoll中有没有已经准备好的g
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if gp := netpoll(false); gp != nil { // non-blocking
			// netpoll returns list of goroutines linked by schedlink.
			injectglist(gp.schedlink.ptr())
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

	// Steal work from other P's.
	// 如果sched.pidle == procs - 1，说明所有的p都是空闲的，无需遍历其他p了
	procs := uint32(gomaxprocs)
	if atomic.Load(&sched.npidle) == procs-1 {
		// Either GOMAXPROCS=1 or everybody, except for us, is idle already.
		// New work can appear from returning syscall/cgocall, network or timers.
		// Neither of that submits to local run queues, so no point in stealing.
		goto stop
	}
	// If number of spinning M's >= number of busy P's, block.
	// This is necessary to prevent excessive CPU consumption
	// when GOMAXPROCS>>1 but the program parallelism is low.
	// 如果寻找p的m的数量，大于有g的p的数量的一般，就不再去寻找了
	if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
		goto stop
	}
	// 设置当前m的自旋状态
	if !_g_.m.spinning {
		_g_.m.spinning = true
		atomic.Xadd(&sched.nmspinning, 1)
	}
	// 开始窃取其他p的待运行g了
	for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			// 从其他的p偷取一般的任务数量，还会随机偷取p的runnext（过分了），偷取部分就不分析了，就是slice的操作而已
			if gp := runqsteal(_p_, allp[enum.position()], stealRunNextG); gp != nil {
				return gp, false
			}
		}
	}

stop:
	// 对all做个镜像备份
	allpSnapshot := allp

	// return P and block
	lock(&sched.lock)

	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
	pidleput(_p_)
	unlock(&sched.lock)

	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		// 设置非自旋状态，因为找p的工作已经结束了
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}
	}

	// check all runqueues once again
	for _, _p_ := range allpSnapshot {
		if !runqempty(_p_) {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				if wasSpinning {
					_g_.m.spinning = true
					atomic.Xadd(&sched.nmspinning, 1)
				}
				goto top
			}
			break
		}
	}
	// poll network
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		if _g_.m.p != 0 {
			throw("findrunnable: netpoll with p")
		}
		if _g_.m.spinning {
			throw("findrunnable: netpoll with spinning")
		}
		gp := netpoll(true) // block until new work is available
		atomic.Store64(&sched.lastpoll, uint64(nanotime()))
		if gp != nil {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				injectglist(gp.schedlink.ptr())
				casgstatus(gp, _Gwaiting, _Grunnable)
				if trace.enabled {
					traceGoUnpark(gp, 0)
				}
				return gp, false
			}
			injectglist(gp)
		}
	}
	stopm()
	goto top
}
```

这里真的是无奈啊，为了寻找一个可运行的g，也是煞费苦心，及时进入了stop 的label，还是不死心，又来了一边寻找。大致寻找过程可以总结为一下几个：

- 从p自己的local队列中获取可运行的g
- 从全局队列中获取可运行的g
- 从netpoll中获取一个已经准备好的g
- 从其他p的local队列中获取可运行的g，随机偷取p的runnext，有点任性
- 无论如何都获取不到的话，就stopm了

##### stopm

stop会把当前m放到空闲列表里面，同时绑定m.nextp 与 m

```go
func stopm() {
	_g_ := getg()
retry:
	lock(&sched.lock)
	// 把当前m放到sched.midle 的空闲列表里
	mput(_g_.m)
	unlock(&sched.lock)
	// 休眠，等待被唤醒
	notesleep(&_g_.m.park)
	noteclear(&_g_.m.park)
	// 绑定p
	acquirep(_g_.m.nextp.ptr())
	_g_.m.nextp = 0
}
```

## 监控

### sysmon

go的监控是依靠函数 sysmon 来完成的，监控主要做一下几件事

- 释放闲置超过5分钟的span物理内存
- 如果超过两分钟没有执行垃圾回收，则强制执行
- 将长时间未处理的netpoll结果添加到任务队列
- 向长时间运行的g进行抢占
- 收回因为syscall而长时间阻塞的p

监控线程并不是时刻在运行的，监控线程首次休眠20us，每次执行完后，增加一倍的休眠时间，但是最多休眠10ms

```go
func sysmon() {
	lock(&sched.lock)
	sched.nmsys++
	checkdead()
	unlock(&sched.lock)

	// If a heap span goes unused for 5 minutes after a garbage collection,
	// we hand it back to the operating system.
	scavengelimit := int64(5 * 60 * 1e9)

	if debug.scavenge > 0 {
		// Scavenge-a-lot for testing.
		forcegcperiod = 10 * 1e6
		scavengelimit = 20 * 1e6
	}

	lastscavenge := nanotime()
	nscavenge := 0

	lasttrace := int64(0)
	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)
	for {
		// 判断当前循环，应该休眠的时间
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		usleep(delay)
		// STW时休眠sysmon
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
			lock(&sched.lock)
			if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
				atomic.Store(&sched.sysmonwait, 1)
				unlock(&sched.lock)
				// Make wake-up period small enough
				// for the sampling to be correct.
				maxsleep := forcegcperiod / 2
				if scavengelimit < forcegcperiod {
					maxsleep = scavengelimit / 2
				}
				shouldRelax := true
				if osRelaxMinNS > 0 {
					next := timeSleepUntil()
					now := nanotime()
					if next-now < osRelaxMinNS {
						shouldRelax = false
					}
				}
				if shouldRelax {
					osRelax(true)
				}
				// 进行休眠
				notetsleep(&sched.sysmonnote, maxsleep)
				if shouldRelax {
					osRelax(false)
				}
				lock(&sched.lock)
				// 唤醒后，清除休眠状态，继续执行
				atomic.Store(&sched.sysmonwait, 0)
				noteclear(&sched.sysmonnote)
				idle = 0
				delay = 20
			}
			unlock(&sched.lock)
		}
		// trigger libc interceptors if needed
		if *cgo_yield != nil {
			asmcgocall(*cgo_yield, nil)
		}
		// poll network if not polled for more than 10ms
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		now := nanotime()
		// 如果netpoll不为空，每隔10ms检查一下是否有ok的
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			// 返回了已经获取到结果的goroutine的列表
			gp := netpoll(false) // non-blocking - returns list of goroutines
			if gp != nil {
				incidlelocked(-1)
				// 把获取到的g的列表加入到全局待运行队列中
				injectglist(gp)
				incidlelocked(1)
			}
		}
		// retake P's blocked in syscalls
		// and preempt long running G's
		// 抢夺syscall长时间阻塞的p和长时间运行的g
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
		// check if we need to force a GC
		// 通过gcTrigger.test() 函数判断是否超过设定的强制触发gc的时间间隔，
		if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
			lock(&forcegc.lock)
			forcegc.idle = 0
			forcegc.g.schedlink = 0
			// 把gc的g加入待运行队列，等待调度运行
			injectglist(forcegc.g)
			unlock(&forcegc.lock)
		}
		// scavenge heap once in a while
		// 判断是否有5分钟未使用的span，有的话，归还给系统
		if lastscavenge+scavengelimit/2 < now {
			mheap_.scavenge(int32(nscavenge), uint64(now), uint64(scavengelimit))
			lastscavenge = now
			nscavenge++
		}
		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
	}
}
```

扫描netpoll，并把g存放到去全局队列比较好理解，跟前面添加p和m的逻辑差不多，但是抢占这里就不是很理解了，你说抢占就抢占，被抢占的g岂不是很没面子，而且怎么抢占呢？

### retake

```go
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
	n := 0
	// Prevent allp slice changes. This lock will be completely
	// uncontended unless we're already stopping the world.
	lock(&allpLock)
	// We can't use a range loop over allp because we may
	// temporarily drop the allpLock. Hence, we need to re-fetch
	// allp each time around the loop.
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		if _p_ == nil {
			// This can happen if procresize has grown
			// allp but not yet created new Ps.
			continue
		}
		pd := &_p_.sysmontick
		s := _p_.status
		if s == _Psyscall {
			// Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
			// pd.syscalltick 即 _p_.sysmontick.syscalltick 只有在sysmon的时候会更新，而 _p_.syscalltick 则会每次都更新，所以，当syscall之后，第一个sysmon检测到的时候并不会抢占，而是第二次开始才会抢占，中间间隔至少有20us，最多会有10ms
			t := int64(_p_.syscalltick)
			if int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
			// On the one hand we don't want to retake Ps if there is no other work to do,
			// but on the other hand we want to retake them eventually
			// because they can prevent the sysmon thread from deep sleep.
			// 是否有空p，有寻找p的m，以及当前的p在syscall之后，有没有超过10ms
			if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			// Drop allpLock so we can take sched.lock.
			unlock(&allpLock)
			// Need to decrement number of idle locked M's
			// (pretending that one more is running) before the CAS.
			// Otherwise the M from which we retake can exit the syscall,
			// increment nmidle and report deadlock.
			incidlelocked(-1)
			// 抢占p，把p的状态转为idle状态
			if atomic.Cas(&_p_.status, s, _Pidle) {
				if trace.enabled {
					traceGoSysBlock(_p_)
					traceProcStop(_p_)
				}
				n++
				_p_.syscalltick++
				// 把当前p移交出去，上面已经分析过了
				handoffp(_p_)
			}
			incidlelocked(1)
			lock(&allpLock)
		} else if s == _Prunning {
			// Preempt G if it's running for too long.
			// 如果p是running状态，如果p下面的g执行太久了，则抢占
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
				continue
			}
			// 判断是否超出10ms, 不超过不抢占
			if pd.schedwhen+forcePreemptNS > now {
				continue
			}
			// 开始抢占
			preemptone(_p_)
		}
	}
	unlock(&allpLock)
	return uint32(n)
}
```

### preemptone

这个函数的注释，作者就表明这种抢占并不是很靠谱😂，我们先看一下实现吧

```go
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	if mp == nil || mp == getg().m {
		return false
	}
	gp := mp.curg
	if gp == nil || gp == mp.g0 {
		return false
	}
	// 标识抢占字段
	gp.preempt = true

	// Every call in a go routine checks for stack overflow by
	// comparing the current stack pointer to gp->stackguard0.
	// Setting gp->stackguard0 to StackPreempt folds
	// preemption into the normal stack overflow check.
	// 更新stackguard0，保证能检测到栈溢
	gp.stackguard0 = stackPreempt
	return true
}
```

在这里，作者会更新   `gp.stackguard0 = stackPreempt`，然后让g误以为栈不够用了，那就只有乖乖的去进行栈扩张，站扩张的话就用调用`newstack` 分配一个新栈，然后把原先的栈的内容拷贝过去，而在 `newstack` 里面有一段如下

```go
if preempt {
	if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
		// Let the goroutine keep running for now.
		// gp->preempt is set, so it will be preempted next time.
		gp.stackguard0 = gp.stack.lo + _StackGuard
		gogo(&gp.sched) // never return
	}
}
```

然后这里就发现g被抢占了，那你栈不够用就有可能是假的，但是管你呢，你再去调度去吧，也不给你扩栈了，虽然作者和雨痕大神都吐槽了一下这个，但是这种抢占方式自动1.5（也可能更早）就一直存在，且稳定运行，就说明还是很牛逼的了

# 总结

在调度器的设置上，最明显的就是复用：g 的free链表， m的free列表， p的free列表，这样就避免了重复创建销毁锁浪费的资源

其次就是多级缓存： 这一块跟内存上的设计思想也是一直的，p一直有一个 g 的待运行队列，自己没有货过多的时候，才会平衡到全局队列，全局队列操作需要锁，则本地操作则不需要，大大减少了锁的创建销毁所消耗的资源

至此，g m p的关系及状态转换大致都讲解完成了，由于对汇编这块比较薄弱，所以基本略过了，右面有机会还是需要多了解一点

# 参考文档

- 《go语言学习笔记》

- [Golang源码探索(二) 协程的实现原理](https://www.cnblogs.com/zkweb/p/7815600.html)

- [【Go源码分析】Go scheduler 源码分析](https://segmentfault.com/a/1190000018777972)