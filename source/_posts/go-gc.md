---
title: 深入理解Go-垃圾回收机制
tags:
  - Go
categories:
  - Go
date: '2019-08-03 18:45'
abbrlink: 13860
---

Go的GC自打出生的时候就开始被人诟病，但是在引入v1.5的三色标记和v1.8的混合写屏障后，正常的GC已经缩短到10us左右，已经变得非常优秀，了不起了，我们接下来探索一下Go的GC的原理吧

<!--more-->

# 三色标记原理

我们首先看一张图，大概就会对 三色标记法 有一个大致的了解：

![](http://note-1253518569.cossh.myqcloud.com/20190803175848.gif)

原理：

1. 首先把所有的对象都放到白色的集合中
2. 从根节点开始遍历对象，遍历到的白色对象从白色集合中放到灰色集合中
3. 遍历灰色集合中的对象，把灰色对象引用的白色集合的对象放入到灰色集合中，同时把遍历过的灰色集合中的对象放到黑色的集合中
4. 循环步骤3，知道灰色集合中没有对象
5. 步骤4结束后，白色集合中的对象就是不可达对象，也就是垃圾，进行回收

# 写屏障

Go在进行三色标记的时候并没有STW，也就是说，此时的对象还是可以进行修改

那么我们考虑一下，下面的情况

![](http://note-1253518569.cossh.myqcloud.com/20190803183838.png)

我们在进行三色标记中扫描灰色集合中，扫描到了对象A，并标记了对象A的所有引用，这时候，开始扫描对象D的引用，而此时，另一个goroutine修改了D->E的引用，变成了如下图所示

![](http://note-1253518569.cossh.myqcloud.com/20190804181356.png)

这样会不会导致E对象就扫描不到了，而被误认为 为白色对象，也就是垃圾

写屏障就是为了解决这样的问题，引入写屏障后，在上述步骤后，E会被认为是存活的，即使后面E被A对象抛弃，E会被在下一轮的GC中进行回收，这一轮GC中是不会对对象E进行回收的

Go1.9中开始启用了混合写屏障，伪代码如下

~~~
writePointer(slot, ptr):
    shade(*slot)
    if any stack is grey:
        shade(ptr)
    *slot = ptr
~~~

混合写屏障会同时标记指针写入目标的"原指针"和“新指针".

标记原指针的原因是, 其他运行中的线程有可能会同时把这个指针的值复制到寄存器或者栈上的本地变量
因为**复制指针到寄存器或者栈上的本地变量不会经过写屏障**, 所以有可能会导致指针不被标记, 试想下面的情况：

```
[go] b = obj
[go] oldx = nil
[gc] scan oldx...
[go] oldx = b.x // 复制b.x到本地变量, 不进过写屏障
[go] b.x = ptr // 写屏障应该标记b.x的原值
[gc] scan b...
如果写屏障不标记原值, 那么oldx就不会被扫描到.
```

标记新指针的原因是, 其他运行中的线程有可能会转移指针的位置, 试想下面的情况:

```
[go] a = ptr
[go] b = obj
[gc] scan b...
[go] b.x = a // 写屏障应该标记b.x的新值
[go] a = nil
[gc] scan a...
如果写屏障不标记新值, 那么ptr就不会被扫描到.
```

混合写屏障可以让GC在并行标记结束后不需要重新扫描各个G的堆栈, 可以减少Mark Termination中的STW时间

除了写屏障外, 在GC的过程中所有新分配的对象都会立刻变为黑色, 在上面的mallocgc函数中可以看到

# 回收流程

GO的GC是并行GC, 也就是GC的大部分处理和普通的go代码是同时运行的, 这让GO的GC流程比较复杂.
首先GC有四个阶段, 它们分别是:

- Sweep Termination: 对未清扫的span进行清扫, 只有上一轮的GC的清扫工作完成才可以开始新一轮的GC
- Mark: 扫描所有根对象, 和根对象可以到达的所有对象, 标记它们不被回收
- Mark Termination: 完成标记工作, 重新扫描部分根对象(要求STW)
- Sweep: 按标记结果清扫span

下图是比较完整的GC流程, 并按颜色对这四个阶段进行了分类:

![](http://note-1253518569.cossh.myqcloud.com/20190803211215.png)

在GC过程中会有两种后台任务(G), 一种是标记用的后台任务, 一种是清扫用的后台任务.
标记用的后台任务会在需要时启动, 可以同时工作的后台任务数量大约是P的数量的25%, 也就是go所讲的让25%的cpu用在GC上的根据.
清扫用的后台任务在程序启动时会启动一个, 进入清扫阶段时唤醒.

目前整个GC流程会进行两次STW(Stop The World), 第一次是Mark阶段的开始, 第二次是Mark Termination阶段.
第一次STW会准备根对象的扫描, 启动写屏障(Write Barrier)和辅助GC(mutator assist).
第二次STW会重新扫描部分根对象, 禁用写屏障(Write Barrier)和辅助GC(mutator assist).
需要注意的是, 不是所有根对象的扫描都需要STW, 例如扫描栈上的对象只需要停止拥有该栈的G.
写屏障的实现使用了Hybrid Write Barrier, 大幅减少了第二次STW的时间.

# 源码分析

## gcStart

~~~go
func gcStart(mode gcMode, trigger gcTrigger) {
	// Since this is called from malloc and malloc is called in
	// the guts of a number of libraries that might be holding
	// locks, don't attempt to start GC in non-preemptible or
	// potentially unstable situations.
	// 判断当前g是否可以抢占，不可抢占时不触发GC
	mp := acquirem()
	if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
		releasem(mp)
		return
	}
	releasem(mp)
	mp = nil

	// Pick up the remaining unswept/not being swept spans concurrently
	//
	// This shouldn't happen if we're being invoked in background
	// mode since proportional sweep should have just finished
	// sweeping everything, but rounding errors, etc, may leave a
	// few spans unswept. In forced mode, this is necessary since
	// GC can be forced at any point in the sweeping cycle.
	//
	// We check the transition condition continuously here in case
	// this G gets delayed in to the next GC cycle.
	// 清扫 残留的未清扫的垃圾
	for trigger.test() && gosweepone() != ^uintptr(0) {
		sweep.nbgsweep++
	}

	// Perform GC initialization and the sweep termination
	// transition.
	semacquire(&work.startSema)
	// Re-check transition condition under transition lock.
	// 判断gcTrriger的条件是否成立
	if !trigger.test() {
		semrelease(&work.startSema)
		return
	}

	// For stats, check if this GC was forced by the user
	// 判断并记录GC是否被强制执行的，runtime.GC()可以被用户调用并强制执行
	work.userForced = trigger.kind == gcTriggerAlways || trigger.kind == gcTriggerCycle

	// In gcstoptheworld debug mode, upgrade the mode accordingly.
	// We do this after re-checking the transition condition so
	// that multiple goroutines that detect the heap trigger don't
	// start multiple STW GCs.
	// 设置gc的mode
	if mode == gcBackgroundMode {
		if debug.gcstoptheworld == 1 {
			mode = gcForceMode
		} else if debug.gcstoptheworld == 2 {
			mode = gcForceBlockMode
		}
	}

	// Ok, we're doing it! Stop everybody else
	semacquire(&worldsema)

	if trace.enabled {
		traceGCStart()
	}
	// 启动后台标记任务
	if mode == gcBackgroundMode {
		gcBgMarkStartWorkers()
	}
	// 重置gc 标记相关的状态
	gcResetMarkState()

	work.stwprocs, work.maxprocs = gomaxprocs, gomaxprocs
	if work.stwprocs > ncpu {
		// This is used to compute CPU time of the STW phases,
		// so it can't be more than ncpu, even if GOMAXPROCS is.
		work.stwprocs = ncpu
	}
	work.heap0 = atomic.Load64(&memstats.heap_live)
	work.pauseNS = 0
	work.mode = mode

	now := nanotime()
	work.tSweepTerm = now
	work.pauseStart = now
	if trace.enabled {
		traceGCSTWStart(1)
	}
	// STW,停止世界
	systemstack(stopTheWorldWithSema)
	// Finish sweep before we start concurrent scan.
	// 先清扫上一轮的垃圾，确保上轮GC完成
	systemstack(func() {
		finishsweep_m()
	})
	// clearpools before we start the GC. If we wait they memory will not be
	// reclaimed until the next GC cycle.
	// 清理 sync.pool sched.sudogcache、sched.deferpool，这里不展开，sync.pool已经说了，剩余的后面的文章会涉及
	clearpools()

	// 增加GC技术
	work.cycles++
	if mode == gcBackgroundMode { // Do as much work concurrently as possible
		gcController.startCycle()
		work.heapGoal = memstats.next_gc

		// Enter concurrent mark phase and enable
		// write barriers.
		//
		// Because the world is stopped, all Ps will
		// observe that write barriers are enabled by
		// the time we start the world and begin
		// scanning.
		//
		// Write barriers must be enabled before assists are
		// enabled because they must be enabled before
		// any non-leaf heap objects are marked. Since
		// allocations are blocked until assists can
		// happen, we want enable assists as early as
		// possible.
		// 设置GC的状态为 gcMark
		setGCPhase(_GCmark)

		// 更新 bgmark 的状态
		gcBgMarkPrepare() // Must happen before assist enable.
		// 计算并排队root 扫描任务，并初始化相关扫描任务状态
		gcMarkRootPrepare()

		// Mark all active tinyalloc blocks. Since we're
		// allocating from these, they need to be black like
		// other allocations. The alternative is to blacken
		// the tiny block on every allocation from it, which
		// would slow down the tiny allocator.
		// 标记 tiny 对象
		gcMarkTinyAllocs()

		// At this point all Ps have enabled the write
		// barrier, thus maintaining the no white to
		// black invariant. Enable mutator assists to
		// put back-pressure on fast allocating
		// mutators.
		// 设置 gcBlackenEnabled 为 1，启用写屏障
		atomic.Store(&gcBlackenEnabled, 1)

		// Assists and workers can start the moment we start
		// the world.
		gcController.markStartTime = now

		// Concurrent mark.
		systemstack(func() {
			now = startTheWorldWithSema(trace.enabled)
		})
		work.pauseNS += now - work.pauseStart
		work.tMark = now
	} else {
		// 非并行模式
		// 记录完成标记阶段的开始时间
		if trace.enabled {
			// Switch to mark termination STW.
			traceGCSTWDone()
			traceGCSTWStart(0)
		}
		t := nanotime()
		work.tMark, work.tMarkTerm = t, t
		work.heapGoal = work.heap0

		// Perform mark termination. This will restart the world.
		// stw,进行标记，清扫并start the world
		gcMarkTermination(memstats.triggerRatio)
	}

	semrelease(&work.startSema)
}
~~~

### gcBgMarkStartWorkers

这个函数准备一些 执行bg mark工作的goroutine，但是这些goroutine并不是立即工作的，而是到等到GC的状态被标记为gcMark 才开始工作，见上个函数的119行

~~~go
func gcBgMarkStartWorkers() {
	// Background marking is performed by per-P G's. Ensure that
	// each P has a background GC G.
	for _, p := range allp {
		if p.gcBgMarkWorker == 0 {
			go gcBgMarkWorker(p)
			// 等待gcBgMarkWorker goroutine 的 bgMarkReady信号再继续
			notetsleepg(&work.bgMarkReady, -1)
			noteclear(&work.bgMarkReady)
		}
	}
}
~~~

#### gcBgMarkWorker

后台标记任务的函数

~~~go
func gcBgMarkWorker(_p_ *p) {
	gp := getg()
	// 用于休眠结束后重新获取p和m
	type parkInfo struct {
		m      muintptr // Release this m on park.
		attach puintptr // If non-nil, attach to this p on park.
	}
	// We pass park to a gopark unlock function, so it can't be on
	// the stack (see gopark). Prevent deadlock from recursively
	// starting GC by disabling preemption.
	gp.m.preemptoff = "GC worker init"
	park := new(parkInfo)
	gp.m.preemptoff = ""
	// 设置park的m和p的信息，留着后面传给gopark，在被gcController.findRunnable唤醒的时候，便于找回
	park.m.set(acquirem())
	park.attach.set(_p_)
	// Inform gcBgMarkStartWorkers that this worker is ready.
	// After this point, the background mark worker is scheduled
	// cooperatively by gcController.findRunnable. Hence, it must
	// never be preempted, as this would put it into _Grunnable
	// and put it on a run queue. Instead, when the preempt flag
	// is set, this puts itself into _Gwaiting to be woken up by
	// gcController.findRunnable at the appropriate time.
	// 让gcBgMarkStartWorkers notetsleepg停止等待并继续及退出
	notewakeup(&work.bgMarkReady)

	for {
		// Go to sleep until woken by gcController.findRunnable.
		// We can't releasem yet since even the call to gopark
		// may be preempted.
		// 让g进入休眠
		gopark(func(g *g, parkp unsafe.Pointer) bool {
			park := (*parkInfo)(parkp)

			// The worker G is no longer running, so it's
			// now safe to allow preemption.
			// 释放当前抢占的m
			releasem(park.m.ptr())

			// If the worker isn't attached to its P,
			// attach now. During initialization and after
			// a phase change, the worker may have been
			// running on a different P. As soon as we
			// attach, the owner P may schedule the
			// worker, so this must be done after the G is
			// stopped.
			// 设置关联p，上面已经设置过了
			if park.attach != 0 {
				p := park.attach.ptr()
				park.attach.set(nil)
				// cas the worker because we may be
				// racing with a new worker starting
				// on this P.
				if !p.gcBgMarkWorker.cas(0, guintptr(unsafe.Pointer(g))) {
					// The P got a new worker.
					// Exit this worker.
					return false
				}
			}
			return true
		}, unsafe.Pointer(park), waitReasonGCWorkerIdle, traceEvGoBlock, 0)

		// Loop until the P dies and disassociates this
		// worker (the P may later be reused, in which case
		// it will get a new worker) or we failed to associate.
		// 检查P的gcBgMarkWorker是否和当前的G一致, 不一致时结束当前的任务
		if _p_.gcBgMarkWorker.ptr() != gp {
			break
		}

		// Disable preemption so we can use the gcw. If the
		// scheduler wants to preempt us, we'll stop draining,
		// dispose the gcw, and then preempt.
		// gopark第一个函数中释放了m，这里再抢占回来
		park.m.set(acquirem())

		if gcBlackenEnabled == 0 {
			throw("gcBgMarkWorker: blackening not enabled")
		}

		startTime := nanotime()
		// 设置gcmark的开始时间
		_p_.gcMarkWorkerStartTime = startTime

		decnwait := atomic.Xadd(&work.nwait, -1)
		if decnwait == work.nproc {
			println("runtime: work.nwait=", decnwait, "work.nproc=", work.nproc)
			throw("work.nwait was > work.nproc")
		}
		// 切换到g0工作
		systemstack(func() {
			// Mark our goroutine preemptible so its stack
			// can be scanned. This lets two mark workers
			// scan each other (otherwise, they would
			// deadlock). We must not modify anything on
			// the G stack. However, stack shrinking is
			// disabled for mark workers, so it is safe to
			// read from the G stack.
			// 设置G的状态为waiting，以便于另一个g扫描它的栈(两个g可以互相扫描对方的栈)
			casgstatus(gp, _Grunning, _Gwaiting)
			switch _p_.gcMarkWorkerMode {
			default:
				throw("gcBgMarkWorker: unexpected gcMarkWorkerMode")
			case gcMarkWorkerDedicatedMode:
				// 专心执行标记工作的模式
				gcDrain(&_p_.gcw, gcDrainUntilPreempt|gcDrainFlushBgCredit)
				if gp.preempt {
					// 被抢占了，把所有本地运行队列中的G放到全局运行队列中
					// We were preempted. This is
					// a useful signal to kick
					// everything out of the run
					// queue so it can run
					// somewhere else.
					lock(&sched.lock)
					for {
						gp, _ := runqget(_p_)
						if gp == nil {
							break
						}
						globrunqput(gp)
					}
					unlock(&sched.lock)
				}
				// Go back to draining, this time
				// without preemption.
				// 继续执行标记工作
				gcDrain(&_p_.gcw, gcDrainNoBlock|gcDrainFlushBgCredit)
			case gcMarkWorkerFractionalMode:
				// 执行标记工作，知道被抢占
				gcDrain(&_p_.gcw, gcDrainFractional|gcDrainUntilPreempt|gcDrainFlushBgCredit)
			case gcMarkWorkerIdleMode:
				// 空闲的时候执行标记工作
				gcDrain(&_p_.gcw, gcDrainIdle|gcDrainUntilPreempt|gcDrainFlushBgCredit)
			}
			// 把G的waiting状态转换到runing状态
			casgstatus(gp, _Gwaiting, _Grunning)
		})

		// If we are nearing the end of mark, dispose
		// of the cache promptly. We must do this
		// before signaling that we're no longer
		// working so that other workers can't observe
		// no workers and no work while we have this
		// cached, and before we compute done.
		// 及时处理本地缓存，上交到全局的队列中
		if gcBlackenPromptly {
			_p_.gcw.dispose()
		}

		// Account for time.
		// 累加耗时
		duration := nanotime() - startTime
		switch _p_.gcMarkWorkerMode {
		case gcMarkWorkerDedicatedMode:
			atomic.Xaddint64(&gcController.dedicatedMarkTime, duration)
			atomic.Xaddint64(&gcController.dedicatedMarkWorkersNeeded, 1)
		case gcMarkWorkerFractionalMode:
			atomic.Xaddint64(&gcController.fractionalMarkTime, duration)
			atomic.Xaddint64(&_p_.gcFractionalMarkTime, duration)
		case gcMarkWorkerIdleMode:
			atomic.Xaddint64(&gcController.idleMarkTime, duration)
		}

		// Was this the last worker and did we run out
		// of work?
		incnwait := atomic.Xadd(&work.nwait, +1)
		if incnwait > work.nproc {
			println("runtime: p.gcMarkWorkerMode=", _p_.gcMarkWorkerMode,
				"work.nwait=", incnwait, "work.nproc=", work.nproc)
			throw("work.nwait > work.nproc")
		}

		// If this worker reached a background mark completion
		// point, signal the main GC goroutine.
		if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
			// Make this G preemptible and disassociate it
			// as the worker for this P so
			// findRunnableGCWorker doesn't try to
			// schedule it.
			// 取消p m的关联
			_p_.gcBgMarkWorker.set(nil)
			releasem(park.m.ptr())

			gcMarkDone()

			// Disable preemption and prepare to reattach
			// to the P.
			//
			// We may be running on a different P at this
			// point, so we can't reattach until this G is
			// parked.
			park.m.set(acquirem())
			park.attach.set(_p_)
		}
	}
}
~~~

##### gcDrain

三色标记的主要实现

gcDrain扫描所有的roots和对象，并表黑灰色对象，知道所有的roots和对象都被标记

~~~go
func gcDrain(gcw *gcWork, flags gcDrainFlags) {
	if !writeBarrier.needed {
		throw("gcDrain phase incorrect")
	}

	gp := getg().m.curg
	// 看到抢占标识是否要返回
	preemptible := flags&gcDrainUntilPreempt != 0
	// 没有任务时是否要等待任务
	blocking := flags&(gcDrainUntilPreempt|gcDrainIdle|gcDrainFractional|gcDrainNoBlock) == 0
	// 是否计算后台的扫描量来减少辅助GC和唤醒等待中的G
	flushBgCredit := flags&gcDrainFlushBgCredit != 0
	// 是否在空闲的时候执行标记任务
	idle := flags&gcDrainIdle != 0
	// 记录初始的已经执行过的扫描任务
	initScanWork := gcw.scanWork

	// checkWork is the scan work before performing the next
	// self-preempt check.
	// 设置对应模式的工作检查函数
	checkWork := int64(1<<63 - 1)
	var check func() bool
	if flags&(gcDrainIdle|gcDrainFractional) != 0 {
		checkWork = initScanWork + drainCheckThreshold
		if idle {
			check = pollWork
		} else if flags&gcDrainFractional != 0 {
			check = pollFractionalWorkerExit
		}
	}

	// Drain root marking jobs.
	// 如果root对象没有扫描完，则扫描
	if work.markrootNext < work.markrootJobs {
		for !(preemptible && gp.preempt) {
			job := atomic.Xadd(&work.markrootNext, +1) - 1
			if job >= work.markrootJobs {
				break
			}
			// 执行root扫描任务
			markroot(gcw, job)
			if check != nil && check() {
				goto done
			}
		}
	}

	// Drain heap marking jobs.
	// 循环直到被抢占
	for !(preemptible && gp.preempt) {
		// Try to keep work available on the global queue. We used to
		// check if there were waiting workers, but it's better to
		// just keep work available than to make workers wait. In the
		// worst case, we'll do O(log(_WorkbufSize)) unnecessary
		// balances.
		if work.full == 0 {
			// 平衡工作，如果全局的标记队列为空，则分一部分工作到全局队列中
			gcw.balance()
		}

		var b uintptr
		if blocking {
			b = gcw.get()
		} else {
			b = gcw.tryGetFast()
			if b == 0 {
				b = gcw.tryGet()
			}
		}
		// 获取任务失败，跳出循环
		if b == 0 {
			// work barrier reached or tryGet failed.
			break
		}
		// 扫描获取的到对象
		scanobject(b, gcw)

		// Flush background scan work credit to the global
		// account if we've accumulated enough locally so
		// mutator assists can draw on it.
		// 如果当前扫描的数量超过了 gcCreditSlack，就把扫描的对象数量加到全局的数量，批量更新
		if gcw.scanWork >= gcCreditSlack {
			atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
			if flushBgCredit {
				gcFlushBgCredit(gcw.scanWork - initScanWork)
				initScanWork = 0
			}
			checkWork -= gcw.scanWork
			gcw.scanWork = 0
			// 如果扫描的对象数量已经达到了 执行下次抢占的目标数量 checkWork， 则调用对应模式的函数
			// idle模式为 pollWork， Fractional模式为 pollFractionalWorkerExit ，在第20行
			if checkWork <= 0 {
				checkWork += drainCheckThreshold
				if check != nil && check() {
					break
				}
			}
		}
	}

	// In blocking mode, write barriers are not allowed after this
	// point because we must preserve the condition that the work
	// buffers are empty.

done:
	// Flush remaining scan work credit.
	if gcw.scanWork > 0 {
		// 把扫描的对象数量添加到全局
		atomic.Xaddint64(&gcController.scanWork, gcw.scanWork)
		if flushBgCredit {
			gcFlushBgCredit(gcw.scanWork - initScanWork)
		}
		gcw.scanWork = 0
	}
}
~~~

###### markroot

这个被用于根对象扫描

~~~go
func markroot(gcw *gcWork, i uint32) {
	// TODO(austin): This is a bit ridiculous. Compute and store
	// the bases in gcMarkRootPrepare instead of the counts.
	baseFlushCache := uint32(fixedRootCount)
	baseData := baseFlushCache + uint32(work.nFlushCacheRoots)
	baseBSS := baseData + uint32(work.nDataRoots)
	baseSpans := baseBSS + uint32(work.nBSSRoots)
	baseStacks := baseSpans + uint32(work.nSpanRoots)
	end := baseStacks + uint32(work.nStackRoots)

	// Note: if you add a case here, please also update heapdump.go:dumproots.
	switch {
	// 释放mcache中的span
	case baseFlushCache <= i && i < baseData:
		flushmcache(int(i - baseFlushCache))
	// 扫描可读写的全局变量
	case baseData <= i && i < baseBSS:
		for _, datap := range activeModules() {
			markrootBlock(datap.data, datap.edata-datap.data, datap.gcdatamask.bytedata, gcw, int(i-baseData))
		}
	// 扫描只读的全局队列
	case baseBSS <= i && i < baseSpans:
		for _, datap := range activeModules() {
			markrootBlock(datap.bss, datap.ebss-datap.bss, datap.gcbssmask.bytedata, gcw, int(i-baseBSS))
		}
	// 扫描Finalizer队列
	case i == fixedRootFinalizers:
		// Only do this once per GC cycle since we don't call
		// queuefinalizer during marking.
		if work.markrootDone {
			break
		}
		for fb := allfin; fb != nil; fb = fb.alllink {
			cnt := uintptr(atomic.Load(&fb.cnt))
			scanblock(uintptr(unsafe.Pointer(&fb.fin[0])), cnt*unsafe.Sizeof(fb.fin[0]), &finptrmask[0], gcw)
		}
	// 释放已经终止的stack
	case i == fixedRootFreeGStacks:
		// Only do this once per GC cycle; preferably
		// concurrently.
		if !work.markrootDone {
			// Switch to the system stack so we can call
			// stackfree.
			systemstack(markrootFreeGStacks)
		}
	// 扫描MSpan.specials
	case baseSpans <= i && i < baseStacks:
		// mark MSpan.specials
		markrootSpans(gcw, int(i-baseSpans))

	default:
		// the rest is scanning goroutine stacks
		// 获取需要扫描的g
		var gp *g
		if baseStacks <= i && i < end {
			gp = allgs[i-baseStacks]
		} else {
			throw("markroot: bad index")
		}

		// remember when we've first observed the G blocked
		// needed only to output in traceback
		status := readgstatus(gp) // We are not in a scan state
		if (status == _Gwaiting || status == _Gsyscall) && gp.waitsince == 0 {
			gp.waitsince = work.tstart
		}

		// scang must be done on the system stack in case
		// we're trying to scan our own stack.
		// 转交给g0进行扫描
		systemstack(func() {
			// If this is a self-scan, put the user G in
			// _Gwaiting to prevent self-deadlock. It may
			// already be in _Gwaiting if this is a mark
			// worker or we're in mark termination.
			userG := getg().m.curg
			selfScan := gp == userG && readgstatus(userG) == _Grunning
			// 如果是扫描自己的，则转换自己的g的状态
			if selfScan {
				casgstatus(userG, _Grunning, _Gwaiting)
				userG.waitreason = waitReasonGarbageCollectionScan
			}

			// TODO: scang blocks until gp's stack has
			// been scanned, which may take a while for
			// running goroutines. Consider doing this in
			// two phases where the first is non-blocking:
			// we scan the stacks we can and ask running
			// goroutines to scan themselves; and the
			// second blocks.
			// 扫描g的栈
			scang(gp, gcw)

			if selfScan {
				casgstatus(userG, _Gwaiting, _Grunning)
			}
		})
	}
}
~~~

###### markRootBlock

根据 ptrmask0，来扫描[b0, b0+n0)区域

~~~go
func markrootBlock(b0, n0 uintptr, ptrmask0 *uint8, gcw *gcWork, shard int) {
	if rootBlockBytes%(8*sys.PtrSize) != 0 {
		// This is necessary to pick byte offsets in ptrmask0.
		throw("rootBlockBytes must be a multiple of 8*ptrSize")
	}

	b := b0 + uintptr(shard)*rootBlockBytes
	// 如果需扫描的block区域，超出b0+n0的区域，直接返回
	if b >= b0+n0 {
		return
	}
	ptrmask := (*uint8)(add(unsafe.Pointer(ptrmask0), uintptr(shard)*(rootBlockBytes/(8*sys.PtrSize))))
	n := uintptr(rootBlockBytes)
	if b+n > b0+n0 {
		n = b0 + n0 - b
	}

	// Scan this shard.
	// 扫描给定block的shard
	scanblock(b, n, ptrmask, gcw)
}
~~~

###### scanblock

~~~go
func scanblock(b0, n0 uintptr, ptrmask *uint8, gcw *gcWork) {
	// Use local copies of original parameters, so that a stack trace
	// due to one of the throws below shows the original block
	// base and extent.
	b := b0
	n := n0

	for i := uintptr(0); i < n; {
		// Find bits for the next word.
		// 找到bitmap中对应的bits
		bits := uint32(*addb(ptrmask, i/(sys.PtrSize*8)))
		if bits == 0 {
			i += sys.PtrSize * 8
			continue
		}
		for j := 0; j < 8 && i < n; j++ {
			if bits&1 != 0 {
				// 如果该地址包含指针
				// Same work as in scanobject; see comments there.
				obj := *(*uintptr)(unsafe.Pointer(b + i))
				if obj != 0 {
					// 如果该地址下找到了对应的对象，标灰
					if obj, span, objIndex := findObject(obj, b, i); obj != 0 {
						greyobject(obj, b, i, span, gcw, objIndex)
					}
				}
			}
			bits >>= 1
			i += sys.PtrSize
		}
	}
}
~~~

###### greyobject

标灰对象其实就是找到对应bitmap，标记存活并扔进队列

~~~go
func greyobject(obj, base, off uintptr, span *mspan, gcw *gcWork, objIndex uintptr) {
	// obj should be start of allocation, and so must be at least pointer-aligned.
	if obj&(sys.PtrSize-1) != 0 {
		throw("greyobject: obj not pointer-aligned")
	}
	mbits := span.markBitsForIndex(objIndex)

	if useCheckmark {
		// 这里是用来debug，确保所有的对象都被正确标识
		if !mbits.isMarked() {
			// 这个对象没有被标记
			printlock()
			print("runtime:greyobject: checkmarks finds unexpected unmarked object obj=", hex(obj), "\n")
			print("runtime: found obj at *(", hex(base), "+", hex(off), ")\n")

			// Dump the source (base) object
			gcDumpObject("base", base, off)

			// Dump the object
			gcDumpObject("obj", obj, ^uintptr(0))

			getg().m.traceback = 2
			throw("checkmark found unmarked object")
		}
		hbits := heapBitsForAddr(obj)
		if hbits.isCheckmarked(span.elemsize) {
			return
		}
		hbits.setCheckmarked(span.elemsize)
		if !hbits.isCheckmarked(span.elemsize) {
			throw("setCheckmarked and isCheckmarked disagree")
		}
	} else {
		if debug.gccheckmark > 0 && span.isFree(objIndex) {
			print("runtime: marking free object ", hex(obj), " found at *(", hex(base), "+", hex(off), ")\n")
			gcDumpObject("base", base, off)
			gcDumpObject("obj", obj, ^uintptr(0))
			getg().m.traceback = 2
			throw("marking free object")
		}

		// If marked we have nothing to do.
		// 对象被正确标记了，无需做其他的操作
		if mbits.isMarked() {
			return
		}
		// mbits.setMarked() // Avoid extra call overhead with manual inlining.
		// 标记对象
		atomic.Or8(mbits.bytep, mbits.mask)
		// If this is a noscan object, fast-track it to black
		// instead of greying it.
		// 如果对象不是指针，则只需要标记，不需要放进队列，相当于直接标黑
		if span.spanclass.noscan() {
			gcw.bytesMarked += uint64(span.elemsize)
			return
		}
	}

	// Queue the obj for scanning. The PREFETCH(obj) logic has been removed but
	// seems like a nice optimization that can be added back in.
	// There needs to be time between the PREFETCH and the use.
	// Previously we put the obj in an 8 element buffer that is drained at a rate
	// to give the PREFETCH time to do its work.
	// Use of PREFETCHNTA might be more appropriate than PREFETCH
	// 判断对象是否被放进队列，没有则放入，标灰步骤完成
	if !gcw.putFast(obj) {
		gcw.put(obj)
	}
}
~~~

###### gcWork.putFast

work有wbuf1 wbuf2两个队列用于保存灰色对象，首先会往wbuf1队列里加入灰色对象，wbuf1满了后，交换wbuf1和wbuf2，这事wbuf2便晋升为wbuf1，继续存放灰色对象，两个队列都满了，则想全局进行申请

putFast这里进尝试将对象放进wbuf1队列中

~~~go
func (w *gcWork) putFast(obj uintptr) bool {
	wbuf := w.wbuf1
	if wbuf == nil {
		// 没有申请缓存队列，返回false
		return false
	} else if wbuf.nobj == len(wbuf.obj) {
		// wbuf1队列满了，返回false
		return false
	}

	// 向未满wbuf1队列中加入对象
	wbuf.obj[wbuf.nobj] = obj
	wbuf.nobj++
	return true
}
~~~

###### gcWork.put

put不仅尝试将对象放入wbuf1，还会再wbuf1满的时候，尝试更换wbuf1 wbuf2的角色，都满的话，则想全局进行申请，并将满的队列上交到全局队列

~~~go
func (w *gcWork) put(obj uintptr) {
	flushed := false
	wbuf := w.wbuf1
	if wbuf == nil {
		// 如果wbuf1不存在，则初始化wbuf1 wbuf2两个队列
		w.init()
		wbuf = w.wbuf1
		// wbuf is empty at this point.
	} else if wbuf.nobj == len(wbuf.obj) {
		// wbuf1满了，更换wbuf1 wbuf2的角色
		w.wbuf1, w.wbuf2 = w.wbuf2, w.wbuf1
		wbuf = w.wbuf1
		if wbuf.nobj == len(wbuf.obj) {
			// 更换角色后，wbuf1也满了，说明两个队列都满了
			// 把 wbuf1上交全局并获取一个空的队列
			putfull(wbuf)
			wbuf = getempty()
			w.wbuf1 = wbuf
			// 设置队列上交的标志位
			flushed = true
		}
	}

	wbuf.obj[wbuf.nobj] = obj
	wbuf.nobj++

	// If we put a buffer on full, let the GC controller know so
	// it can encourage more workers to run. We delay this until
	// the end of put so that w is in a consistent state, since
	// enlistWorker may itself manipulate w.
	// 此时全局已经有标记满的队列，GC controller选择调度更多work进行工作
	if flushed && gcphase == _GCmark {
		gcController.enlistWorker()
	}
}
~~~

到这里，接下来，我们继续分析gcDrain里面的函数，追踪一下，我们标灰的对象是如何被标黑的

###### gcw.balance()

继续分析 gcDrain的58行，balance work是什么

~~~go
func (w *gcWork) balance() {
	if w.wbuf1 == nil {
		// 这里wbuf1 wbuf2队列还没有初始化
		return
	}
	// 如果wbuf2不为空，则上交到全局，并获取一个空岛队列给wbuf2
	if wbuf := w.wbuf2; wbuf.nobj != 0 {
		putfull(wbuf)
		w.wbuf2 = getempty()
	} else if wbuf := w.wbuf1; wbuf.nobj > 4 {
		// 把未满的wbuf1分成两半，并把其中一半上交的全局队列
		w.wbuf1 = handoff(wbuf)
	} else {
		return
	}
	// We flushed a buffer to the full list, so wake a worker.
	// 这里，全局队列有满的队列了，其他work可以工作了
	if gcphase == _GCmark {
		gcController.enlistWorker()
	}
}
~~~

###### gcw.get()

继续分析 gcDrain的63行，这里就是首先从本地的队列获取一个对象，如果本地队列的wbuf1没有，尝试从wbuf2获取，如果两个都没有，则尝试从全局队列获取一个满的队列，并获取一个对象

~~~go
func (w *gcWork) get() uintptr {
	wbuf := w.wbuf1
	if wbuf == nil {
		w.init()
		wbuf = w.wbuf1
		// wbuf is empty at this point.
	}
	if wbuf.nobj == 0 {
		// wbuf1空了，更换wbuf1 wbuf2的角色
		w.wbuf1, w.wbuf2 = w.wbuf2, w.wbuf1
		wbuf = w.wbuf1
		// 原wbuf2也是空的，尝试从全局队列获取一个满的队列
		if wbuf.nobj == 0 {
			owbuf := wbuf
			wbuf = getfull()
			// 获取不到，则返回
			if wbuf == nil {
				return 0
			}
			// 把空的队列上传到全局空队列，并把获取的满的队列，作为自身的wbuf1
			putempty(owbuf)
			w.wbuf1 = wbuf
		}
	}

	// TODO: This might be a good place to add prefetch code

	wbuf.nobj--
	return wbuf.obj[wbuf.nobj]
}
~~~

` gcw.tryGet()` ` gcw.tryGetFast()` 逻辑差不多，相对比较简单，就不继续分析了

###### scanobject

我们继续分析到 gcDrain 的L76，这里已经获取到了b，开始消费队列

~~~go
func scanobject(b uintptr, gcw *gcWork) {
	// Find the bits for b and the size of the object at b.
	//
	// b is either the beginning of an object, in which case this
	// is the size of the object to scan, or it points to an
	// oblet, in which case we compute the size to scan below.
	// 获取b对应的bits
	hbits := heapBitsForAddr(b)
	// 获取b所在的span
	s := spanOfUnchecked(b)
	n := s.elemsize
	if n == 0 {
		throw("scanobject n == 0")
	}
	// 对象过大，则切割后再扫描，maxObletBytes为128k
	if n > maxObletBytes {
		// Large object. Break into oblets for better
		// parallelism and lower latency.
		if b == s.base() {
			// It's possible this is a noscan object (not
			// from greyobject, but from other code
			// paths), in which case we must *not* enqueue
			// oblets since their bitmaps will be
			// uninitialized.
			// 如果不是指针，直接标记返回，相当于标黑了
			if s.spanclass.noscan() {
				// Bypass the whole scan.
				gcw.bytesMarked += uint64(n)
				return
			}

			// Enqueue the other oblets to scan later.
			// Some oblets may be in b's scalar tail, but
			// these will be marked as "no more pointers",
			// so we'll drop out immediately when we go to
			// scan those.
			// 按maxObletBytes切割后放入到 队列
			for oblet := b + maxObletBytes; oblet < s.base()+s.elemsize; oblet += maxObletBytes {
				if !gcw.putFast(oblet) {
					gcw.put(oblet)
				}
			}
		}

		// Compute the size of the oblet. Since this object
		// must be a large object, s.base() is the beginning
		// of the object.
		n = s.base() + s.elemsize - b
		if n > maxObletBytes {
			n = maxObletBytes
		}
	}

	var i uintptr
	for i = 0; i < n; i += sys.PtrSize {
		// Find bits for this word.
		// 获取到对应的bits
		if i != 0 {
			// Avoid needless hbits.next() on last iteration.
			hbits = hbits.next()
		}
		// Load bits once. See CL 22712 and issue 16973 for discussion.
		bits := hbits.bits()
		// During checkmarking, 1-word objects store the checkmark
		// in the type bit for the one word. The only one-word objects
		// are pointers, or else they'd be merged with other non-pointer
		// data into larger allocations.
		if i != 1*sys.PtrSize && bits&bitScan == 0 {
			break // no more pointers in this object
		}
		// 不是指针，继续
		if bits&bitPointer == 0 {
			continue // not a pointer
		}

		// Work here is duplicated in scanblock and above.
		// If you make changes here, make changes there too.
		obj := *(*uintptr)(unsafe.Pointer(b + i))

		// At this point we have extracted the next potential pointer.
		// Quickly filter out nil and pointers back to the current object.
		if obj != 0 && obj-b >= n {
			// Test if obj points into the Go heap and, if so,
			// mark the object.
			//
			// Note that it's possible for findObject to
			// fail if obj points to a just-allocated heap
			// object because of a race with growing the
			// heap. In this case, we know the object was
			// just allocated and hence will be marked by
			// allocation itself.
			// 找到指针对应的对象，并标灰
			if obj, span, objIndex := findObject(obj, b, i); obj != 0 {
				greyobject(obj, b, i, span, gcw, objIndex)
			}
		}
	}
	gcw.bytesMarked += uint64(n)
	gcw.scanWork += int64(i)
}
~~~

综上，我们可以发现，标灰就是标记并放进队列，标黑就是标记，所以当灰色对象从队列中取出后，我们就可以认为这个对象是黑色对象了

至此，gcDrain的标记工作分析完成，我们继续回到gcBgMarkWorker分析

##### gcMarkDone

gcMarkDone会将mark1阶段进入到mark2阶段， mark2阶段进入到mark termination阶段

mark1阶段： 包括所有root标记，全局缓存队列和本地缓存队列

mark2阶段：本地缓存队列会被禁用

~~~go
func gcMarkDone() {
top:
	semacquire(&work.markDoneSema)

	// Re-check transition condition under transition lock.
	if !(gcphase == _GCmark && work.nwait == work.nproc && !gcMarkWorkAvailable(nil)) {
		semrelease(&work.markDoneSema)
		return
	}

	// Disallow starting new workers so that any remaining workers
	// in the current mark phase will drain out.
	//
	// TODO(austin): Should dedicated workers keep an eye on this
	// and exit gcDrain promptly?
	// 禁止新的标记任务
	atomic.Xaddint64(&gcController.dedicatedMarkWorkersNeeded, -0xffffffff)
	prevFractionalGoal := gcController.fractionalUtilizationGoal
	gcController.fractionalUtilizationGoal = 0

	// 如果gcBlackenPromptly表名需要所有本地缓存队列立即上交到全局队列，并禁用本地缓存队列
	if !gcBlackenPromptly {
		// Transition from mark 1 to mark 2.
		//
		// The global work list is empty, but there can still be work
		// sitting in the per-P work caches.
		// Flush and disable work caches.

		// Disallow caching workbufs and indicate that we're in mark 2.
		// 禁用本地缓存队列，进入mark2阶段
		gcBlackenPromptly = true

		// Prevent completion of mark 2 until we've flushed
		// cached workbufs.
		atomic.Xadd(&work.nwait, -1)

		// GC is set up for mark 2. Let Gs blocked on the
		// transition lock go while we flush caches.
		semrelease(&work.markDoneSema)
		// 切换到g0执行，本地缓存上传到全局的操作
		systemstack(func() {
			// Flush all currently cached workbufs and
			// ensure all Ps see gcBlackenPromptly. This
			// also blocks until any remaining mark 1
			// workers have exited their loop so we can
			// start new mark 2 workers.
			forEachP(func(_p_ *p) {
				wbBufFlush1(_p_)
				_p_.gcw.dispose()
			})
		})

		// Check that roots are marked. We should be able to
		// do this before the forEachP, but based on issue
		// #16083 there may be a (harmless) race where we can
		// enter mark 2 while some workers are still scanning
		// stacks. The forEachP ensures these scans are done.
		//
		// TODO(austin): Figure out the race and fix this
		// properly.
		// 检查所有的root是否都被标记了
		gcMarkRootCheck()

		// Now we can start up mark 2 workers.
		atomic.Xaddint64(&gcController.dedicatedMarkWorkersNeeded, 0xffffffff)
		gcController.fractionalUtilizationGoal = prevFractionalGoal

		incnwait := atomic.Xadd(&work.nwait, +1)
		// 如果没有更多的任务，则执行第二次调用，从mark2阶段转换到mark termination阶段
		if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
			// This loop will make progress because
			// gcBlackenPromptly is now true, so it won't
			// take this same "if" branch.
			goto top
		}
	} else {
		// Transition to mark termination.
		now := nanotime()
		work.tMarkTerm = now
		work.pauseStart = now
		getg().m.preemptoff = "gcing"
		if trace.enabled {
			traceGCSTWStart(0)
		}
		systemstack(stopTheWorldWithSema)
		// The gcphase is _GCmark, it will transition to _GCmarktermination
		// below. The important thing is that the wb remains active until
		// all marking is complete. This includes writes made by the GC.

		// Record that one root marking pass has completed.
		work.markrootDone = true

		// Disable assists and background workers. We must do
		// this before waking blocked assists.
		atomic.Store(&gcBlackenEnabled, 0)

		// Wake all blocked assists. These will run when we
		// start the world again.
		// 唤醒所有的辅助GC
		gcWakeAllAssists()

		// Likewise, release the transition lock. Blocked
		// workers and assists will run when we start the
		// world again.
		semrelease(&work.markDoneSema)

		// endCycle depends on all gcWork cache stats being
		// flushed. This is ensured by mark 2.
		// 计算下一次gc出发的阈值
		nextTriggerRatio := gcController.endCycle()

		// Perform mark termination. This will restart the world.
		// start the world，并进入完成阶段
		gcMarkTermination(nextTriggerRatio)
	}
}
~~~

###### gcMarkTermination

结束标记，并进行清扫等工作

~~~go
func gcMarkTermination(nextTriggerRatio float64) {
	// World is stopped.
	// Start marktermination which includes enabling the write barrier.
	atomic.Store(&gcBlackenEnabled, 0)
	gcBlackenPromptly = false
	// 设置GC的阶段标识
	setGCPhase(_GCmarktermination)

	work.heap1 = memstats.heap_live
	startTime := nanotime()

	mp := acquirem()
	mp.preemptoff = "gcing"
	_g_ := getg()
	_g_.m.traceback = 2
	gp := _g_.m.curg
	// 设置当前g的状态为waiting状态
	casgstatus(gp, _Grunning, _Gwaiting)
	gp.waitreason = waitReasonGarbageCollection

	// Run gc on the g0 stack. We do this so that the g stack
	// we're currently running on will no longer change. Cuts
	// the root set down a bit (g0 stacks are not scanned, and
	// we don't need to scan gc's internal state).  We also
	// need to switch to g0 so we can shrink the stack.
	systemstack(func() {
		// 通过g0扫描当前g的栈
		gcMark(startTime)
		// Must return immediately.
		// The outer function's stack may have moved
		// during gcMark (it shrinks stacks, including the
		// outer function's stack), so we must not refer
		// to any of its variables. Return back to the
		// non-system stack to pick up the new addresses
		// before continuing.
	})

	systemstack(func() {
		work.heap2 = work.bytesMarked
		if debug.gccheckmark > 0 {
			// Run a full stop-the-world mark using checkmark bits,
			// to check that we didn't forget to mark anything during
			// the concurrent mark process.
			// 如果启用了gccheckmark，则检查所有可达对象是否都有标记
			gcResetMarkState()
			initCheckmarks()
			gcMark(startTime)
			clearCheckmarks()
		}

		// marking is complete so we can turn the write barrier off
		// 设置gc的阶段标识，GCoff时会关闭写屏障
		setGCPhase(_GCoff)
		// 开始清扫
		gcSweep(work.mode)

		if debug.gctrace > 1 {
			startTime = nanotime()
			// The g stacks have been scanned so
			// they have gcscanvalid==true and gcworkdone==true.
			// Reset these so that all stacks will be rescanned.
			gcResetMarkState()
			finishsweep_m()

			// Still in STW but gcphase is _GCoff, reset to _GCmarktermination
			// At this point all objects will be found during the gcMark which
			// does a complete STW mark and object scan.
			setGCPhase(_GCmarktermination)
			gcMark(startTime)
			setGCPhase(_GCoff) // marking is done, turn off wb.
			gcSweep(work.mode)
		}
	})

	_g_.m.traceback = 0
	casgstatus(gp, _Gwaiting, _Grunning)

	if trace.enabled {
		traceGCDone()
	}

	// all done
	mp.preemptoff = ""

	if gcphase != _GCoff {
		throw("gc done but gcphase != _GCoff")
	}

	// Update GC trigger and pacing for the next cycle.
	// 更新下次出发gc的增长比
	gcSetTriggerRatio(nextTriggerRatio)

	// Update timing memstats
	// 更新用时
	now := nanotime()
	sec, nsec, _ := time_now()
	unixNow := sec*1e9 + int64(nsec)
	work.pauseNS += now - work.pauseStart
	work.tEnd = now
	atomic.Store64(&memstats.last_gc_unix, uint64(unixNow)) // must be Unix time to make sense to user
	atomic.Store64(&memstats.last_gc_nanotime, uint64(now)) // monotonic time for us
	memstats.pause_ns[memstats.numgc%uint32(len(memstats.pause_ns))] = uint64(work.pauseNS)
	memstats.pause_end[memstats.numgc%uint32(len(memstats.pause_end))] = uint64(unixNow)
	memstats.pause_total_ns += uint64(work.pauseNS)

	// Update work.totaltime.
	sweepTermCpu := int64(work.stwprocs) * (work.tMark - work.tSweepTerm)
	// We report idle marking time below, but omit it from the
	// overall utilization here since it's "free".
	markCpu := gcController.assistTime + gcController.dedicatedMarkTime + gcController.fractionalMarkTime
	markTermCpu := int64(work.stwprocs) * (work.tEnd - work.tMarkTerm)
	cycleCpu := sweepTermCpu + markCpu + markTermCpu
	work.totaltime += cycleCpu

	// Compute overall GC CPU utilization.
	totalCpu := sched.totaltime + (now-sched.procresizetime)*int64(gomaxprocs)
	memstats.gc_cpu_fraction = float64(work.totaltime) / float64(totalCpu)

	// Reset sweep state.
	// 重置清扫的状态
	sweep.nbgsweep = 0
	sweep.npausesweep = 0

	// 如果是强制开启的gc，标识增加
	if work.userForced {
		memstats.numforcedgc++
	}

	// Bump GC cycle count and wake goroutines waiting on sweep.
	// 统计执行GC的次数然后唤醒等待清扫的G
	lock(&work.sweepWaiters.lock)
	memstats.numgc++
	injectglist(work.sweepWaiters.head.ptr())
	work.sweepWaiters.head = 0
	unlock(&work.sweepWaiters.lock)

	// Finish the current heap profiling cycle and start a new
	// heap profiling cycle. We do this before starting the world
	// so events don't leak into the wrong cycle.
	mProf_NextCycle()
	// start the world
	systemstack(func() { startTheWorldWithSema(true) })

	// Flush the heap profile so we can start a new cycle next GC.
	// This is relatively expensive, so we don't do it with the
	// world stopped.
	mProf_Flush()

	// Prepare workbufs for freeing by the sweeper. We do this
	// asynchronously because it can take non-trivial time.
	prepareFreeWorkbufs()

	// Free stack spans. This must be done between GC cycles.
	systemstack(freeStackSpans)

	// Print gctrace before dropping worldsema. As soon as we drop
	// worldsema another cycle could start and smash the stats
	// we're trying to print.
	if debug.gctrace > 0 {
		util := int(memstats.gc_cpu_fraction * 100)

		var sbuf [24]byte
		printlock()
		print("gc ", memstats.numgc,
			" @", string(itoaDiv(sbuf[:], uint64(work.tSweepTerm-runtimeInitTime)/1e6, 3)), "s ",
			util, "%: ")
		prev := work.tSweepTerm
		for i, ns := range []int64{work.tMark, work.tMarkTerm, work.tEnd} {
			if i != 0 {
				print("+")
			}
			print(string(fmtNSAsMS(sbuf[:], uint64(ns-prev))))
			prev = ns
		}
		print(" ms clock, ")
		for i, ns := range []int64{sweepTermCpu, gcController.assistTime, gcController.dedicatedMarkTime + gcController.fractionalMarkTime, gcController.idleMarkTime, markTermCpu} {
			if i == 2 || i == 3 {
				// Separate mark time components with /.
				print("/")
			} else if i != 0 {
				print("+")
			}
			print(string(fmtNSAsMS(sbuf[:], uint64(ns))))
		}
		print(" ms cpu, ",
			work.heap0>>20, "->", work.heap1>>20, "->", work.heap2>>20, " MB, ",
			work.heapGoal>>20, " MB goal, ",
			work.maxprocs, " P")
		if work.userForced {
			print(" (forced)")
		}
		print("\n")
		printunlock()
	}

	semrelease(&worldsema)
	// Careful: another GC cycle may start now.

	releasem(mp)
	mp = nil

	// now that gc is done, kick off finalizer thread if needed
	// 如果不是并行GC，则让当前M开始调度
	if !concurrentSweep {
		// give the queued finalizers, if any, a chance to run
		Gosched()
	}
}
~~~

###### goSweep

清扫任务

~~~go
func gcSweep(mode gcMode) {
	if gcphase != _GCoff {
		throw("gcSweep being done but phase is not GCoff")
	}

	lock(&mheap_.lock)
	// sweepgen在每次GC之后都会增长2，每次GC之后sweepSpans的角色都会互换
	mheap_.sweepgen += 2
	mheap_.sweepdone = 0
	if mheap_.sweepSpans[mheap_.sweepgen/2%2].index != 0 {
		// We should have drained this list during the last
		// sweep phase. We certainly need to start this phase
		// with an empty swept list.
		throw("non-empty swept list")
	}
	mheap_.pagesSwept = 0
	unlock(&mheap_.lock)
	// 如果不是并行GC，或者强制GC
	if !_ConcurrentSweep || mode == gcForceBlockMode {
		// Special case synchronous sweep.
		// Record that no proportional sweeping has to happen.
		lock(&mheap_.lock)
		mheap_.sweepPagesPerByte = 0
		unlock(&mheap_.lock)
		// Sweep all spans eagerly.
		// 清扫所有的span
		for sweepone() != ^uintptr(0) {
			sweep.npausesweep++
		}
		// Free workbufs eagerly.
		// 释放所有的 workbufs
		prepareFreeWorkbufs()
		for freeSomeWbufs(false) {
		}
		// All "free" events for this mark/sweep cycle have
		// now happened, so we can make this profile cycle
		// available immediately.
		mProf_NextCycle()
		mProf_Flush()
		return
	}

	// Background sweep.
	lock(&sweep.lock)
	// 唤醒后台清扫任务,也就是 bgsweep 函数，清扫流程跟上面非并行清扫差不多
	if sweep.parked {
		sweep.parked = false
		ready(sweep.g, 0, true)
	}
	unlock(&sweep.lock)
}
~~~

###### sweepone

接下来我们就分析一下sweepone 清扫的流程

~~~go
func sweepone() uintptr {
	_g_ := getg()
	sweepRatio := mheap_.sweepPagesPerByte // For debugging

	// increment locks to ensure that the goroutine is not preempted
	// in the middle of sweep thus leaving the span in an inconsistent state for next GC
	_g_.m.locks++
	// 检查是否已经完成了清扫
	if atomic.Load(&mheap_.sweepdone) != 0 {
		_g_.m.locks--
		return ^uintptr(0)
	}
	// 增加清扫的worker数量
	atomic.Xadd(&mheap_.sweepers, +1)

	npages := ^uintptr(0)
	sg := mheap_.sweepgen
	for {
		// 循环获取需要清扫的span
		s := mheap_.sweepSpans[1-sg/2%2].pop()
		if s == nil {
			atomic.Store(&mheap_.sweepdone, 1)
			break
		}
		if s.state != mSpanInUse {
			// This can happen if direct sweeping already
			// swept this span, but in that case the sweep
			// generation should always be up-to-date.
			if s.sweepgen != sg {
				print("runtime: bad span s.state=", s.state, " s.sweepgen=", s.sweepgen, " sweepgen=", sg, "\n")
				throw("non in-use span in unswept list")
			}
			continue
		}
		// sweepgen == h->sweepgen - 2, 表示这个span需要清扫
		// sweepgen == h->sweepgen - 1, 表示这个span正在被清扫
		// 这是里确定span的状态及尝试转换span的状态
		if s.sweepgen != sg-2 || !atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			continue
		}
		npages = s.npages
		// 单个span的清扫
		if !s.sweep(false) {
			// Span is still in-use, so this returned no
			// pages to the heap and the span needs to
			// move to the swept in-use list.
			npages = 0
		}
		break
	}

	// Decrement the number of active sweepers and if this is the
	// last one print trace information.
	// 当前worker清扫任务完成，更新sweepers的数量
	if atomic.Xadd(&mheap_.sweepers, -1) == 0 && atomic.Load(&mheap_.sweepdone) != 0 {
		if debug.gcpacertrace > 0 {
			print("pacer: sweep done at heap size ", memstats.heap_live>>20, "MB; allocated ", (memstats.heap_live-mheap_.sweepHeapLiveBasis)>>20, "MB during sweep; swept ", mheap_.pagesSwept, " pages at ", sweepRatio, " pages/byte\n")
		}
	}
	_g_.m.locks--
	return npages
}
~~~

###### mspan.sweep

~~~go
func (s *mspan) sweep(preserve bool) bool {
	// It's critical that we enter this function with preemption disabled,
	// GC must not start while we are in the middle of this function.
	_g_ := getg()
	if _g_.m.locks == 0 && _g_.m.mallocing == 0 && _g_ != _g_.m.g0 {
		throw("MSpan_Sweep: m is not locked")
	}
	sweepgen := mheap_.sweepgen
	// 只有正在清扫中状态的span才可以正常执行
	if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
		print("MSpan_Sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
		throw("MSpan_Sweep: bad span state")
	}

	if trace.enabled {
		traceGCSweepSpan(s.npages * _PageSize)
	}
	// 先更新清扫的page数
	atomic.Xadd64(&mheap_.pagesSwept, int64(s.npages))

	spc := s.spanclass
	size := s.elemsize
	res := false

	c := _g_.m.mcache
	freeToHeap := false

	// The allocBits indicate which unmarked objects don't need to be
	// processed since they were free at the end of the last GC cycle
	// and were not allocated since then.
	// If the allocBits index is >= s.freeindex and the bit
	// is not marked then the object remains unallocated
	// since the last GC.
	// This situation is analogous to being on a freelist.

	// Unlink & free special records for any objects we're about to free.
	// Two complications here:
	// 1. An object can have both finalizer and profile special records.
	//    In such case we need to queue finalizer for execution,
	//    mark the object as live and preserve the profile special.
	// 2. A tiny object can have several finalizers setup for different offsets.
	//    If such object is not marked, we need to queue all finalizers at once.
	// Both 1 and 2 are possible at the same time.
	specialp := &s.specials
	special := *specialp
	// 判断在special中的对象是否存活，是否至少有一个finalizer，释放没有finalizer的对象，把有finalizer的对象组成队列
	for special != nil {
		// A finalizer can be set for an inner byte of an object, find object beginning.
		objIndex := uintptr(special.offset) / size
		p := s.base() + objIndex*size
		mbits := s.markBitsForIndex(objIndex)
		if !mbits.isMarked() {
			// This object is not marked and has at least one special record.
			// Pass 1: see if it has at least one finalizer.
			hasFin := false
			endOffset := p - s.base() + size
			for tmp := special; tmp != nil && uintptr(tmp.offset) < endOffset; tmp = tmp.next {
				if tmp.kind == _KindSpecialFinalizer {
					// Stop freeing of object if it has a finalizer.
					mbits.setMarkedNonAtomic()
					hasFin = true
					break
				}
			}
			// Pass 2: queue all finalizers _or_ handle profile record.
			for special != nil && uintptr(special.offset) < endOffset {
				// Find the exact byte for which the special was setup
				// (as opposed to object beginning).
				p := s.base() + uintptr(special.offset)
				if special.kind == _KindSpecialFinalizer || !hasFin {
					// Splice out special record.
					y := special
					special = special.next
					*specialp = special
					freespecial(y, unsafe.Pointer(p), size)
				} else {
					// This is profile record, but the object has finalizers (so kept alive).
					// Keep special record.
					specialp = &special.next
					special = *specialp
				}
			}
		} else {
			// object is still live: keep special record
			specialp = &special.next
			special = *specialp
		}
	}

	if debug.allocfreetrace != 0 || raceenabled || msanenabled {
		// Find all newly freed objects. This doesn't have to
		// efficient; allocfreetrace has massive overhead.
		mbits := s.markBitsForBase()
		abits := s.allocBitsForIndex(0)
		for i := uintptr(0); i < s.nelems; i++ {
			if !mbits.isMarked() && (abits.index < s.freeindex || abits.isMarked()) {
				x := s.base() + i*s.elemsize
				if debug.allocfreetrace != 0 {
					tracefree(unsafe.Pointer(x), size)
				}
				if raceenabled {
					racefree(unsafe.Pointer(x), size)
				}
				if msanenabled {
					msanfree(unsafe.Pointer(x), size)
				}
			}
			mbits.advance()
			abits.advance()
		}
	}

	// Count the number of free objects in this span.
	// 获取需要释放的alloc对象的总数
	nalloc := uint16(s.countAlloc())
	// 如果sizeclass为0，却分配的总数量为0，则释放到mheap
	if spc.sizeclass() == 0 && nalloc == 0 {
		s.needzero = 1
		freeToHeap = true
	}
	nfreed := s.allocCount - nalloc
	if nalloc > s.allocCount {
		print("runtime: nelems=", s.nelems, " nalloc=", nalloc, " previous allocCount=", s.allocCount, " nfreed=", nfreed, "\n")
		throw("sweep increased allocation count")
	}

	s.allocCount = nalloc
	// 判断span是否empty
	wasempty := s.nextFreeIndex() == s.nelems
	// 重置freeindex
	s.freeindex = 0 // reset allocation index to start of span.
	if trace.enabled {
		getg().m.p.ptr().traceReclaimed += uintptr(nfreed) * s.elemsize
	}

	// gcmarkBits becomes the allocBits.
	// get a fresh cleared gcmarkBits in preparation for next GC
	// 重置 allocBits为 gcMarkBits
	s.allocBits = s.gcmarkBits
	// 重置 gcMarkBits
	s.gcmarkBits = newMarkBits(s.nelems)

	// Initialize alloc bits cache.
	// 更新allocCache
	s.refillAllocCache(0)

	// We need to set s.sweepgen = h.sweepgen only when all blocks are swept,
	// because of the potential for a concurrent free/SetFinalizer.
	// But we need to set it before we make the span available for allocation
	// (return it to heap or mcentral), because allocation code assumes that a
	// span is already swept if available for allocation.
	if freeToHeap || nfreed == 0 {
		// The span must be in our exclusive ownership until we update sweepgen,
		// check for potential races.
		if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
			print("MSpan_Sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
			throw("MSpan_Sweep: bad span state after sweep")
		}
		// Serialization point.
		// At this point the mark bits are cleared and allocation ready
		// to go so release the span.
		atomic.Store(&s.sweepgen, sweepgen)
	}

	if nfreed > 0 && spc.sizeclass() != 0 {
		c.local_nsmallfree[spc.sizeclass()] += uintptr(nfreed)
		// 把span释放到mcentral上
		res = mheap_.central[spc].mcentral.freeSpan(s, preserve, wasempty)
		// MCentral_FreeSpan updates sweepgen
	} else if freeToHeap {
		// 这里是大对象的span释放，与117行呼应
		// Free large span to heap

		// NOTE(rsc,dvyukov): The original implementation of efence
		// in CL 22060046 used SysFree instead of SysFault, so that
		// the operating system would eventually give the memory
		// back to us again, so that an efence program could run
		// longer without running out of memory. Unfortunately,
		// calling SysFree here without any kind of adjustment of the
		// heap data structures means that when the memory does
		// come back to us, we have the wrong metadata for it, either in
		// the MSpan structures or in the garbage collection bitmap.
		// Using SysFault here means that the program will run out of
		// memory fairly quickly in efence mode, but at least it won't
		// have mysterious crashes due to confused memory reuse.
		// It should be possible to switch back to SysFree if we also
		// implement and then call some kind of MHeap_DeleteSpan.
		if debug.efence > 0 {
			s.limit = 0 // prevent mlookup from finding this span
			sysFault(unsafe.Pointer(s.base()), size)
		} else {
			// 把sapn释放到mheap上
			mheap_.freeSpan(s, 1)
		}
		c.local_nlargefree++
		c.local_largefree += size
		res = true
	}
	if !res {
		// The span has been swept and is still in-use, so put
		// it on the swept in-use list.
		// 如果span未释放到mcentral或mheap，表示span仍然处于in-use状态
		mheap_.sweepSpans[sweepgen/2%2].push(s)
	}
	return res
}
~~~

ok，至此Go的GC流程已经分析完成了，结合最上面开始的图，可能会容易理解一点



# 参考文档

1. [Golang源码探索(三) GC的实现原理](https://www.cnblogs.com/zkweb/p/7880099.html)
2. 《Go语言学习笔记》
3. [一张图了解三色标记法](http://idiotsky.top/2017/08/16/gc-three-color/)