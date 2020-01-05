---
title: 深入理解Go-map原理剖析
tags:
  - Go
categories:
  - Go
date: '2019-10-08 20:04'
abbrlink: 4823
---

在使用map的过程中，有两个问题是经常会遇到的：读写冲突和遍历无序性。为什么会这样呢，底层是怎么实现的呢？带着这两个问题，我简单的了解了一下map的增删改查及遍历的实现。

<!--more-->


# 结构

## hmap

~~~go
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // 有效数据的长度# live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8 // 用于记录hashmap的状态
	B         uint8  // 2^B = buckets的数量log_2 of # of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // 随机的hash种子

	buckets    unsafe.Pointer // buckets数组array of 2^B Buckets. may be nil if count==0.
	oldbuckets unsafe.Pointer // 老的buctedts数据，map增长的时候会用到
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // 额外的bmap数组optional fields
}
~~~

## mapextra

~~~go
 type mapextra struct {
	// If both key and value do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and value do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
~~~



## bmap

~~~go
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit more complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
~~~

## stringStruct

~~~go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
~~~

## hiter

map遍历时用到的结构，startBucket+offset设定了开始遍历的地址，保证map遍历的无序性

~~~go
type hiter struct {
  // key的指针
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/internal/gc/range.go).
  // 当前value的指针
	value       unsafe.Pointer // Must be in second position (see cmd/internal/gc/range.go).
	t           *maptype
  // 指向map的指针
	h           *hmap
  // 指向buckets的指针
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
  // 指向当前遍历的bucket的指针
	bptr        *bmap          // current bucket
  // 指向map.extra.overflow
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
  // 指向map.extra.oldoverflow
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
  // 开始遍历的bucket的索引
	startBucket uintptr        // bucket iteration started at
  // 开始遍历bucket上的偏移量
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}
~~~







![](http://note-1253518569.cossh.myqcloud.com/20190930145115.png)

这里的keys和values、*overflow三个变量在结构体中并没有体现，但是在源码过程中，一直有为他们预留位置，所以这里的示意图中就展示出来了，keys和values其实8个长度的数组

# demo

我们简单写个demo，通过`go tool` 来分析一下底层所对应的函数

~~~go
func main() {
	m := make(map[interface{}]interface{}, 16)
	m["111"] = 1
	m["222"] = 2
	m["444"] = 4
	_ = m["444"]
	_, _ = m["444"]
	delete(m, "444")

	for range m {
	}
}
~~~



~~~
▶ go tool objdump -s "main.main" main | grep CALL
  main.go:4             0x455c74                e8f761fbff              CALL runtime.makemap(SB)                
  main.go:5             0x455ce1                e8da6dfbff              CALL runtime.mapassign(SB)              
  main.go:6             0x455d7b                e8406dfbff              CALL runtime.mapassign(SB)              
  main.go:7             0x455e15                e8a66cfbff              CALL runtime.mapassign(SB)              
  main.go:8             0x455e88                e89363fbff              CALL runtime.mapaccess1(SB)             
  main.go:9             0x455ec4                e84766fbff              CALL runtime.mapaccess2(SB)             
  main.go:10            0x455f00                e85b72fbff              CALL runtime.mapdelete(SB)              
  main.go:12            0x455f28                e804a7ffff              CALL 0x450631                           
  main.go:12            0x455f53                e8b875fbff              CALL runtime.mapiterinit(SB)            
  main.go:12            0x455f75                e88677fbff              CALL runtime.mapiternext(SB)            
  main.go:7             0x455f8f                e81c9cffff              CALL runtime.gcWriteBarrier(SB)         
  main.go:6             0x455f9c                e80f9cffff              CALL runtime.gcWriteBarrier(SB)         
  main.go:5             0x455fa9                e8029cffff              CALL runtime.gcWriteBarrier(SB)         
  main.go:3             0x455fb3                e8f87dffff              CALL runtime.morestack_noctxt(SB) 
~~~



# 初始化

## makemap

makemap创建一个hmap结构体，并赋予这个变量一些初始的属性

~~~go
func makemap(t *maptype, hint int, h *hmap) *hmap {
  // 首先判断map的大小是否合适
	if hint < 0 || hint > int(maxSliceCap(t.bucket.size)) {
		hint = 0
	}

	// initialize Hmap
  // 初始化hmap结构
	if h == nil {
		h = new(hmap)
	}
  // 生成一个随机的hash种子
	h.hash0 = fastrand()

	// find size parameter which will hold the requested # of elements
  // 根据hint，也就是map预设的长度，确定B的大小，以使map的装载系数在正常范围内，扩容那块再细讲
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
  // 如果B==0，则赋值的时候进行惰性分配，如果B！=0，则分配对应数量的buckets
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
~~~

##makeBucketArray

makeBucketArray初始化了map所需的buckets，最少分配2^b个buckets

~~~go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := bucketShift(b)
	nbuckets := base
	// 如果b，也就是map比较大的情况，则多分配点数组，给nextOverflow使用
	if b >= 4 {
    // 计算应该多分配的buckets数量
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}
	// 如果不是 dirtyalloc,新分配map空间时，dirtyalloc为nil
	if dirtyalloc == nil {
    // 申请buckets数组
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
		if t.bucket.kind&kindNoPointers == 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}
  // 判断是否多申请了buckets，多申请的buckets放在nextOverflow里面以备后用
	if base != nbuckets {
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
~~~

初始化的过程到此就结束了，比较简单，就是根据初始化的大小，确定buckets的数量，并分配内存等

# 查找（mapaccess）

在上面的`go tool` 分析过程中可以发现

- _ = m["444"] 对应 `mapaccess1`
- _, _ = m["444"] 对应 `mapaccess2`

两个函数的逻辑大致相同，我们以`mapaccess1`为例来分析

## mapaccess

~~~go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
  // 如果h还没有实例化，或者还没有值，返回零值
	if h == nil || h.count == 0 {
		return unsafe.Pointer(&zeroVal[0])
	}
  // 判断当前map是否处于 写 的过程中，读写冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
  // 根据初始化生产的hash随机种子hash0，计算key的hash值
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
  // 根据key的hash值，计算出对应的bucket的位置，计算过程后面图示
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  // 扩容的过程中，oldbuckets不为空，所以这时候，这时候需要判断，目标bucket是否已经迁移完成了，扩容的时候细讲
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
    // 如果目标bucket在扩容中还没有迁移，则到oldbuckets中找目标bucket
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
  // 计算出key的tophash，用于比对
	top := tophash(hash)
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
      // 如果tophash不一致，key肯定不同，继续寻找下一个
			if b.tophash[i] != top {
				continue
			}
      // tophash一直，需要判断key是否一致
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey {
				k = *((*unsafe.Pointer)(k))
			}
      // key也是相同的，则返回对应的value
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				if t.indirectvalue {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}
~~~

### overflow

这个函数就是找bmap的overflow的地址，通过结构图中可以看出，找到bmap结构体的最后一个指针占用的内存单元就是overflow指向的下一个bmap的地址了

```go
func (b *bmap) overflow(t *maptype) *bmap {
	return *(**bmap)(add(unsafe.Pointer(b), uintptr(t.bucketsize)-sys.PtrSize))
}
```



上面的逻辑比较简单，但是在这里有几个问题需要解决

1. bucket（bmap结构体）是怎么确定的
2. tophash是怎么确定的
3. key和value的地址为什么是通过偏移来计算的

先放一下buckets和bmap的放大图

![](http://note-1253518569.cossh.myqcloud.com/20190930144229.png)

1. bucket（bmap结构体）是怎么确定的

   ```go
   bucket := hash & bucketMask(h.B)
   b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
   ```

   加入B=5，则说明buckets的数量为2^5 = 32，则取hash的末5位，来计算出目标bucket的索引，图中计算出索引为6，所以，在buckets上偏移6个bucket大小的地址，即可找到对应的bucket

2. tophash是怎么确定的

   ```go
   func tophash(hash uintptr) uint8 {
   	top := uint8(hash >> (sys.PtrSize*8 - 8))
   	if top < minTopHash {
   		top += minTopHash
   	}
   	return top
   }
   ```

   每个bucket的tophash数组的长度为8，所以，这里直接去hash值的前8位计算出来数值，既是tophash了

3. key和value的地址为什么是通过偏移来计算的

   ```go
   k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
   val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
   ```

   根据最开始的数据结构分析和上面的bmap图示，可以看出bmap中所有的key是放在一起的，所有的value是放在一起的，dataoffset是tophash[8]所占用的大小，所以，key所在的地址也就是 b的地址+dataOffset的偏移+对应的索引i*key的大小，同理value是排列在key的后面的

# 插入

## mapassign

~~~go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	// map并发读写的处理，直接抛异常
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
  // 根据map的hash种子 hash0，计算key的hash值
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write.
	h.flags |= hashWriting
  // 如果map没有buckets，就分配（make(map)不指定map长度的时候就会惰性分配buckets）
	if h.buckets == nil {
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
  // 根据计算出的hash值，来确定应该插入的bucket在buckets中的索引
	bucket := hash & bucketMask(h.B)
  // 判断是否在扩容map，growWork是来完成扩容操作的
	if h.growing() {
		growWork(t, h, bucket)
	}
  // 确认bucket的地址
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
  // 根据计算出hash二进制前八位的值，作为tophash使用
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
      // 循环遍历tophash数组，如果数组的索引位置为空，先拿过来使用
			if b.tophash[i] != top {
				if b.tophash[i] == empty && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
				continue
			}
      // 找到了tophash数组中找到了当前key的tophash一致的情况
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
      // 如果key是指针，获取指针对应的数据
			if t.indirectkey {
				k = *((*unsafe.Pointer)(k))
			}
      // 判断这两个key是否相同，不同继续寻找
			if !alg.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate {
				typedmemmove(t.key, k, key)
			}
      // 根据i找到value应该存放的位置，可以结合结构图中bmap的数据结构来理解
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
    // buckets中没有找到空余的位置或者相同的key，则到overflow中查找
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
  // 判断是否需要扩容
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	// inerti==nil，表示map的buckets都满了，则需要新加一个overflow挂载到map和对应的bmap下
	if inserti == nil {
		// all current buckets are full, allocate a new one.
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/value at insert position
  // 存储key value到指定的位置
	if t.indirectkey {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
  // 修改map的flags
	h.flags &^= hashWriting
	if t.indirectvalue {
		val = *((*unsafe.Pointer)(val))
	}
	return val
}
~~~

### setoverflow

~~~go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
	var ovf *bmap
  // 先去找一下预先分配的有没有剩余的overflow
	if h.extra != nil && h.extra.nextOverflow != nil {
		// We have preallocated overflow buckets available.
		// See makeBucketArray for more details.
    // 预先分配的有，直接使用预先分配的，然后更新一下 下一个可以用overflow => nextOverflow
		ovf = h.extra.nextOverflow
		if ovf.overflow(t) == nil {
			// We're not at the end of the preallocated overflow buckets. Bump the pointer.
			h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.bucketsize)))
		} else {
			// This is the last preallocated overflow bucket.
			// Reset the overflow pointer on this bucket,
			// which was set to a non-nil sentinel value.
			ovf.setoverflow(t, nil)
			h.extra.nextOverflow = nil
		}
	} else {
		ovf = (*bmap)(newobject(t.bucket))
	}
  // 增加noverflow
	h.incrnoverflow()
	if t.bucket.kind&kindNoPointers != 0 {
		h.createOverflow()
		*h.extra.overflow = append(*h.extra.overflow, ovf)
	}
  // 把当前overflow，挂载到bmap的overflow链表后面
	b.setoverflow(t, ovf)
	return ovf
}
~~~

overflow指向的就是一个bmap结构，而bmap结构的最后一个地址，存储的是overflow的地址，通过bmap.overflow可以将bmap的所有overflow串联起来，hmap.extra.nextOverflow也是一样的逻辑

# 扩容

在`mapassign`函数中可以看到，扩容发生的情况有两种

~~~go
overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)
~~~

1. 超过设定的负载值
2. 有太多的overflow

先来看一下这两个函数

## overLoadFactor

~~~go
func overLoadFactor(count int, B uint8) bool {
  // loadFactorNum = 13; loadFactorDen = 2
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
~~~

`uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)` 可以简化为 `count / (2^B) > 6.5`， 这个6.5便是代表loadFactor的负载系数

##tooManyOverflowBuckets

~~~go
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	// If the threshold is too low, we do extraneous work.
	// If the threshold is too high, maps that grow and shrink can hold on to lots of unused memory.
	// "too many" means (approximately) as many overflow buckets as regular buckets.
	// See incrnoverflow for more details.
	if B > 15 {
		B = 15
	}
	// The compiler doesn't see here that B < 16; mask B to generate shorter shift code.
	return noverflow >= uint16(1)<<(B&15)
}
~~~

通过判断noverflow的数量来判断overflow是否太多



我们理解一下这两种情况扩容的原因

1. 超过设定的负载值

   根据key查找的过程中，根据末B位确定bucket，高8位确定tophash，但是查找tophash的过程中，是需要遍历整个bucket的，所以，最优的情况是每个bucket只存储一个key，这样就达到了hash的O(1)的查找效率，但是空间却大大的浪费了；如果所有的key都存储到了一个bucket里面面，就退变成了链表，查找效率就变成了O(n)，所以装载系数就是为了平衡查找效率和存储空间的，当装载系数过大，就需要增加bucket了，来提高查找效率，即增量扩容

2. 有太多的overflow

   当bucket的空位全部填满的时候，装载系数就达到了8，为什么还会有tooManyOverflowBuckets的判断呢，map不仅有增加还有删除的操作，当某一个bucket的空位填满后，开始填充到overflow里面，这时候再删除bucket里面的数据，其实整个过程很有可能并没有触发 超过负载扩容机制的，（因为有较多的buckets），但是查找overflow的数据，就首先要遍历bucket的数据，这个就是无用功了，查找效率就低了，这时候需要不增加bucket数量的扩容，也就是等量扩容



扩容的工作是由`hashGrow`开始的，但是真正进行迁移工作的是`evacuate`， 由`growWork`进行d调用；在每一次的maassign和mapdelete的时候，会判断这个map是否正在进行扩容操作，如果是的，就迁移当前的bucket；所以，map的扩容并不是一蹴而就的，而是一个循序渐进的过程

## hashGrow

~~~go
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
  // 判断是等量扩容还是增量扩容
	bigger := uint8(1)
	if !overLoadFactor(h.count+1, h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
  // 为map根据新的B（h.B+bigger为新的h.B）重新分配新的buckets和overflow
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
  // 更新hmap相关的属性
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0
	// 将老的map的extra和nextOverflow更新到新的map结构下面
	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}
~~~

 `hashGrow` 这个前菜已经准备完成了，接下来就交给`growWork`和 `evacuate`两个函数来完成的

### growWork

~~~go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}
~~~

###evacuate

讲hmap中的一个bucket搬移到新的buckets中，老的bucket里key与新的buckets中位置的对应，同样参考map的查找过程

这里如何判断这个bucket是否已经搬移过了呢，主要就是依据`evacuated`函数来判断

~~~go
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > empty && h < minTopHash
}
~~~

看了源码就发现原理很简单，就是对tophash[0]值的判断，那么肯定是在搬移之后设置的这个值，我们通过`evacuate`函数l哎一探究竟吧

~~~go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
  // 判断是否搬移过
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
    // 吧bucket原先对应的索引赋值给x
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))
		// 如果是增量扩容，扩容后的bucket有变，假如以B=5为例，B+1= 6，这时候去倒数6位计算bucket的索引，但是倒数第6位只能是0或者1，也就是说索引只能是，x或y（x+newbit）,这里计算出来y，以备后用
		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.v = add(y.k, bucketCnt*uintptr(t.keysize))
		}
		// 进行搬移
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
        // 空的跳过
				if top == empty {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/value to bucket x or bucket y).
          // 判断hash计算出来，是使用x还是y，等量扩容是使用x
					hash := t.key.alg.hash(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey && !t.key.alg.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination
				// 如果目标的bucket已经满了，则新建overflow，挂载到bucket上，并使用这个overflow
				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
        // 拷贝key value，设置tophash数组的对应索引的值
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy value
				}
				if t.indirectvalue {
					*(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
				} else {
					typedmemmove(t.elem, dst.v, v)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or value arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		// Unlink the overflow buckets & clear key/value to help GC.
		if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
~~~

扩容是逐步进行的，一次搬运一个bucket

我们以原先的B=5为例，现在增量扩容后B=6，但是hash的倒数第6位只能是0或1，也就是说，如果原先计算出来的bucket索引为6的话，即 00110，那么新的bucket对应的索引只能是 100110（6+2^5）或 000110（6），x对应的就是6，y对应的就是（6+2^5）；如果是等量扩容，那么索引肯定就是不变的，这时候就不需要y了

找到对应的新的bucket之后，按顺序依次存放就ok了

# 删除

## mapdelete

删除的逻辑比较简单，根据key查找，找到就清空key和value及tophash

~~~go
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if h == nil || h.count == 0 {
		return
	}
  // 读写冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
	// 下面一大片的计算hash，查找bucket，查到bucket里面的key，逻辑一样，就不重复了
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write (delete).
	h.flags |= hashWriting

	bucket := hash & bucketMask(h.B)
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !alg.equal(key, k2) {
				continue
			}
			// Only clear key if there are pointers in it.
      // 这里找到了key，如果key是指针，设为nil，否则清空key对应内存的数据
			if t.indirectkey {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.kind&kindNoPointers == 0 {
				memclrHasPointers(k, t.key.size)
			}
      // 同理删除v
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			if t.indirectvalue {
				*(*unsafe.Pointer)(v) = nil
			} else if t.elem.kind&kindNoPointers == 0 {
				memclrHasPointers(v, t.elem.size)
			} else {
				memclrNoHeapPointers(v, t.elem.size)
			}
      // 把tophash设置为0，并更新count属性
			b.tophash[i] = empty
			h.count--
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}
~~~

# 遍历

按一般的思维来考虑，遍历值需要遍历buckets数组里面的每个bucket以及bucket下挂的overflow链表即可，但是map存在扩容的情况，这样就会导致遍历的难度增大了，我们看一下go是怎么实现的

根据`go tool` 的分析，我们可以简单看一下遍历时的流程信息

![](http://note-1253518569.cossh.myqcloud.com/20191008172252.png)

## mapiterinit

~~~go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
  // 设置iter的属性
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.kind&kindNoPointers != 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
  // 随机生成一个种子，并根据这个随机种子计算出startBucket和offset，保证遍历的随机性
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}
	// 开始遍历
	mapiternext(it)
}
~~~

## mapiternext

~~~go
func mapiternext(it *hiter) {
	h := it.h
	if raceenabled {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket
	alg := t.key.alg

next:
  // b==nil说明bucket.overflow链表已经遍历完成了，遍历下一个bucket
	if b == nil {
    // 遍历到了开始的bucket，而且startBucket被遍历过了，则说明整个map遍历完成了
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.value = nil
			return
		}
    // 如果hmap正在扩容，则判断当前遍历的bucket是否搬移完了，搬移完了，使用新得bucket，否则使用oldbucket
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
    // 遍历到了数组末尾，从数组头继续遍历
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
  // 遍历当前bucket或者bucket.overflow里面的数据
	for ; i < bucketCnt; i++ {
    // 通过offset与i，确定正在遍历的bucket的tophash的索引
		offi := (i + it.offset) & (bucketCnt - 1)
		if b.tophash[offi] == empty || b.tophash[offi] == evacuatedEmpty {
			continue
		}
    // 根据偏移量i，确定key和value的地址
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey {
			k = *((*unsafe.Pointer)(k))
		}
		v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.valuesize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
      // 说明增量扩容中，需要进一步判断
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey || alg.equal(k, k) {
        // 数据还没有从oldbucket迁移到新的bucket里面，判断这个key重新计算后是否与oldbucket的索引一致，不一致则跳过
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := alg.hash(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey || alg.equal(k, k)) {
      // 这里的数据不是正在扩容中的数据，可以直接使用
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectvalue {
				v = *((*unsafe.Pointer)(v))
			}
			it.value = v
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
      // 在遍历开始之后，这个map进行了扩容，数据可能不正确，重新查找获取一下
			rk, rv := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.value = rv
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
  // 遍历bucket.overflow链表
	b = b.overflow(t)
	i = 0
	goto next
}
~~~

整体思路如下：

1. 首先从buckets数组中，随机确定一个索引，作为startBucket，然后确定offset偏移量，作为起始key的地址
2. 遍历当前bucket及bucket.overflow，判断当前bucket是否正在扩容中，如果是则跳转到3，否则跳转到4
3. 加入原先的buckets为0，1，那么扩容后的新的buckets为0，1，2，3，此时我们遍历到了buckets[0]， 发现这个bucket正在扩容，那么找到bucket[0]所对应的oldbuckets[0]，遍历里面的key，这时候是遍历所有的吗？当然不是，而是仅仅遍历那些key经过hash，可以散列到bucket[0]里面的部分key；同理，当遍历到bucket[2]的时候，发现bucket正在扩容，找到oldbuckets[0]，然后遍历里面可以散列到bucket[2]的那些key
4. 遍历当前这个bucket即可
5. 继续遍历bucket下面的overflow链表
6. 如果遍历到了startBucket，说明遍历完了，结束遍历



# 参考文章

- [《深度解密Go语言之 map》](https://juejin.im/post/5ce4dd5ae51d4558936a9fde#heading-9)