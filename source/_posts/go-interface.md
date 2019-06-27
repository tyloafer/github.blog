---
title: 深入理解Go的interface
tags:
  - Go
categories:
  - Go
date: '2019-04-25 18:45'
abbrlink: 2647
---

简单介绍一下我对interface的理解，涉及指针类型转换，指针运算，interface结构及反射等

<!-- more -->

## Unsafe.Pointer 和 uintptr

### 指针类型转换
我们一般使用 `*T` 来表示一个指向类型T变量的指针， 类似于 `*int` 表示 int型指针。但是指针类型并不能进行类型转换
~~~
func main() {
	var i *int
	p := 10
	i = &p

	m := (*float64)(i)
}
~~~

~~~
cannot convert i (type *int) to type *float64
~~~
如上，在进行类型转换的时候就会报上述的错误

### unsafe.Pointer
unsafe.Pointer 是特别定义的一种指针类型（译注：类似C语言中的void类型的指针），它可以包含任意类型变量的地址，是各个指针类型转换的桥梁，所以上述的指针类型转换就可以通过 unsafe.Pointer 实现
~~~
func main() {
	var i *int
	p := 10
	i = &p

	m := (*float64)(unsafe.Pointer(i))
}
~~~

unsafe.Pointer的最终实现其实就是一个 *int
~~~
type ArbitraryType int

type Pointer *ArbitraryType
~~~

当然，pointer的转换也是有规则的
> A safe pointer can be explicitly converted to an unsafe pointer, and vice versa.（安全指针可以显式转换为不安全指针，反之亦然。）
> An uintptr value can be explicitly converted to an unsafe pointer, and vice versa. But please note, a nil unsafe.Pointer shouldn't be converted to uintptr and back with arithmetic.（uintptr值可以显式转换为不安全指针，反之亦然。）


### uintptr
一个可以容纳任何指针类型的整型，可以进行指针运算，而Go的指针是不支持运算的，这个也是上述转换的第二条规则的意义，转换成 unintptr 进行指针运算，然后再转换为对应的 类型的指针，以显示变量

~~~
type User struct {
	Name string
	Age int
}

func main() {
	u := new(User)
	name :=(*string)(unsafe.Pointer(u))
	*name = "test"

	age := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(u)) + unsafe.Offsetof(u.Age)))
	*age = 18
	fmt.Println(u)
}
~~~
思路: 如果想对Name和 Age进行赋值，那首先应该先拿到相应的地址，然后对地址内容修改，所以我们可以分别定义一个 *string 和 *int 分别指向 Name 和 Age的地址
对于 u这个结构体，u的首地址就是Name的地址，但是u是结构体指针，所以只需要将u转换成 *string 即可
对于 Age，我们已经拿到了Name的地址，在此基础上进行偏移即可，也就是 unsafe.Offsetof(u.Age) 的偏移量， 但是 `Pointer`不能进行指针运算，随意需要将 `Pointer` 转换成 `uintptr`


## interface
### interface的坑
~~~
package main

import (
	"fmt"
)

type Err struct{}

func (err *Err) Error() string {
	return ""
}

func main() {
	var i interface{}
	i = nil
	if i == nil {
		fmt.Println("i is nil")
	}

	var a *int
	a = nil
	if a == nil {
		fmt.Println("a is nil")
	}

	a1 := testInterface()
	if a1 == nil {
		fmt.Println("test interface return nil")
	} else {
		fmt.Println("test interface return not nil")
	}

	a2 := testInterfacePointer()
	if a2 == nil {
		fmt.Println("test interface pointer return nil")
	} else {
		fmt.Println("test interface pointer return not nil")
	}
}

func testInterface() error {
	return nil
}

func testInterfacePointer() error {
	var p *Err = nil
	return p
}
~~~

以上面一个代码为例，打印结果如下

~~~
i is nil
a is nil
test interface return nil
test interface pointer return not nil

~~~

前面三个结果比较好理解，但是为什么 `testInterfacePointer` 返回结果不是 `nil`
我们用gdb调试，看一下

~~~
i = {_type = 0x0, data = 0x0}
a = 0x0
a1 = {tab = 0x0, data = 0x0}
a2 = {tab = 0x4c8c40 <*main.Err,error>, data = 0x0}

~~~
可以看出来，a2 结构体中的 `tab` 不是 `0x0`， 所以，运行结果判断 `a2 != nil`

为什么i的结构体包含的是 `_type` `data`, 但是 a1 和 a2包含的结构体内容是 `tab` `data`

### interface结构体
#### eface (empty interface)
顾名思义，就是空的interfae, 使用 `interface{}` 声明的都是 empty interface
~~~
type eface struct {
	_type *_type
	data  unsafe.Pointer
}

type _type struct {
	size       uintptr // Size returns the number of bytes needed to store a value of the given type，it is analogous to unsafe.Sizeof.可通过 reflec
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32 // 类型的hash值，所有的type会存放在hashType表中，以hash为索引
	tflag      tflag //  额外类型信息标志
	align      uint8 // 此类型type的对齐
	fieldalign uint8 // 这个type的结构体字段的对齐
	kind       uint8 // 类型编号 定义于runtime/typekind.go
	alg        *typeAlg
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff // 类型名字的偏移
	ptrToThis typeOff
}
~~~

#### iface (带函数的interface)
~~~
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype // interface的类型
	_type *_type // 同上的type
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.， fun在栈上偏移地址，因为 fun都是指针，所以长度都是4个字节， 找到fun的首地址后，每偏移4个地址便存储一个函数地址
}

type interfacetype struct {
	typ     _type
	pkgpath name // 包信息
	mhdr    []imethod // method slice，这里是排序后的，方便检测 struct是否继承了 interface
}

type imethod struct {
	name nameOff // 函数名字的偏移，这里是int32,后面查找的时候会根据这个数值，计算出偏移量， 存放在 firstmoduledata 这个符号表中
	ityp typeOff // type类型的偏移量，对应interface的_type
}
~~~

- `size`: 存储一个指定类型的值需要的bytes， 可通过 reflect.TypeOf(int8(1)).Size() 或 unsafe.Sizeof(x interface{})来获取

    ~~~
    So(unsafe.Sizeof(true), ShouldEqual, 1)
    So(unsafe.Sizeof(int8(0)), ShouldEqual, 1)
    So(unsafe.Sizeof(int16(0)), ShouldEqual, 2)
    So(unsafe.Sizeof(int32(0)), ShouldEqual, 4)
    So(unsafe.Sizeof(int64(0)), ShouldEqual, 8)
    So(unsafe.Sizeof(int(0)), ShouldEqual, 8)
    So(unsafe.Sizeof(float32(0)), ShouldEqual, 4)
    So(unsafe.Sizeof(float64(0)), ShouldEqual, 8)
    So(unsafe.Sizeof(""), ShouldEqual, 16)
    So(unsafe.Sizeof("hello world"), ShouldEqual, 16)
    So(unsafe.Sizeof([]int{}), ShouldEqual, 24)
    So(unsafe.Sizeof([]int{1, 2, 3}), ShouldEqual, 24)
    So(unsafe.Sizeof([3]int{1, 2, 3}), ShouldEqual, 24)
    So(unsafe.Sizeof(map[string]string{}), ShouldEqual, 8)
    So(unsafe.Sizeof(map[string]string{"1": "one", "2": "two"}), ShouldEqual, 8)
    So(unsafe.Sizeof(struct{}{}), ShouldEqual, 0)
    ~~~
- `_type`: 就是表示结构体的类型，这些类型会被存放在 firstmoduledata 这个符号表中，反射其实就是根据 _type 里面偏移量或者索引，到对应地方获取我们认知的数据并返回
- `data`: 就是指向 数据存放的地址的指针，例如上面的 i interface{} = 3, 其实就是在堆或栈上分配一个空间存放这些数据，然后 data中存放这个地址（这里可以优化，如果 数据占用空间比指针还小的话，可以直接存放在data里面，以节约空间）
- `iface:inter`: 其实是定义的interface的类型
- `iface:inter:mhdr`: 这里是interface里面定义的函数，某个struct是否继承了这个interface，就是通过 struct的函数与 mhdr里面的函数进行对比，（这里为了提高比对的效率，这个slice会进行排序，将原本 m*n 的效率提升至 m + n）
- `fun`: 
    这里是 结构体函数的平铺，如下代码所示，首先我们存储好函数后，把函数的地址，追加到 `fun[0]` 的地址指向的空间后面
    ~~~
    ...
    ifn := typ.textOff(t.ifn)
    
    *(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
    ...
    ~~~
    同时，我们不会用到 fun的长度，而是根据指定的索引去获取我们需要的函数，也就是如下使用
    ~~~
    fn = unsafe.Pointer(&iface.itab.fun[i])
    ~~~
    所以在这里，我们不需要知道 fun 的长度，及fun的结束地址


### 反射
~~~
type Err struct {
	Msg string
}

func (err *Err) Error() string {
	return ""
}

func main() {
	err := &Err{"error"}
	
	// get method
	method, _ := reflect.TypeOf(err).MethodByName("Error")
	fmt.Println(method)

	// get method by method index
	method0 := reflect.TypeOf(err).Method(0)
	fmt.Println(method0)

	// get kind
	kind := reflect.TypeOf(err).Kind()
	fmt.Println("unknow kind ", kind)

	// change value
	reflect.ValueOf(err).Elem().Field(0).SetString("test")
	fmt.Println(*err)
}
~~~
返回结果：
~~~
{Error  func(*main.Err) string <func(*main.Err) string Value> 0}
{Error  func(*main.Err) string <func(*main.Err) string Value> 0}
unknow kind  ptr
{test}
~~~
接下来，根据上面的例子，分别追踪一下 `TypeOf` `MethodByName` `Method` `Method` `Elem` `Field` `SetString` 这些函数在我们的执行流程中华，都做了些什么，加深一下对interface的理解

#### 相关结构体
这里是
~~~
type Value struct {
	// typ holds the type of the value represented by a Value.
	typ *rtype

	// Pointer-valued data or, if flagIndir is set, pointer to data.
	// Valid when either flagIndir is set or typ.pointers() is true.
	ptr unsafe.Pointer

	// flag holds metadata about the value.
	// The lowest bits are flag bits:
	//	- flagStickyRO: obtained via unexported not embedded field, so read-only
	//	- flagEmbedRO: obtained via unexported embedded field, so read-only
	//	- flagIndir: val holds a pointer to the data
	//	- flagAddr: v.CanAddr is true (implies flagIndir)
	//	- flagMethod: v is a method value.
	// The next five bits give the Kind of the value.
	// This repeats typ.Kind() except for method values.
	// The remaining 23+ bits give a method number for method values.
	// If flag.kind() != Func, code can assume that flagMethod is unset.
	// If ifaceIndir(typ), code can assume that flagIndir is set.
	flag
}

type Type interface {
	Align() int
	FieldAlign() int
	Method(int) Method
	NumMethod() int
	Name() string
	PkgPath() string
	Size() uintptr
	String() string
	Kind() Kind
	Implements(u Type) bool
	AssignableTo(u Type) bool
	ConvertibleTo(u Type) bool
	Comparable() bool
	Bits() int
	ChanDir() ChanDir
	IsVariadic() bool	Elem() Type
	Field(i int) StructField
	FieldByIndex(index []int) StructField
	FieldByName(name string) (StructField, bool)
	FieldByNameFunc(match func(string) bool) (StructField, bool)
	In(i int) Type
	Key() Type
	Len() int
	NumField() int
	NumIn() int
	NumOut() int
	common() *rtype
	uncommon() *uncommonType
}

type emptyInterface struct {
	typ  *rtype
	word unsafe.Pointer
}

type interfaceType struct {
	rtype
	pkgPath name      // import path
	methods []imethod // sorted by hash
}

type rtype struct {
	size       uintptr
	ptrdata    uintptr  // number of bytes in the type that can contain pointers
	hash       uint32   // hash of type; avoids computation in hash tables
	tflag      tflag    // extra type information flags
	align      uint8    // alignment of variable with this type
	fieldAlign uint8    // alignment of struct field with this type
	kind       uint8    // enumeration for C
	alg        *typeAlg // algorithm table
	gcdata     *byte    // garbage collection data
	str        nameOff  // string form
	ptrToThis  typeOff  // type for pointer to this type, may be zero
}

type name struct {
	bytes *byte
}

~~~

#### TypeOf(err).MethodByName("Error")
**TypeOf**
~~~
func TypeOf(i interface{}) Type {
	eface := *(*emptyInterface)(unsafe.Pointer(&i))
	return toType(eface.typ)
}

func toType(t *rtype) Type {
	if t == nil {
		return nil
	}
	return t
}
~~~
根据上面的逻辑，`TypeOf` 就是把 interface{} 转成了 emptyInterface， 然后 返回了 `rtype`结构， 在 `func toType(t *rtype) Type` 中，要求返回的是 `Type` 接口，也就是说 `rtype` 继承了 `Type`，那我们就可以继续使用`Type`中的函数了

**MethodByName**
~~~

func (t *rtype) MethodByName(name string) (m Method, ok bool) {
	if t.Kind() == Interface {
		tt := (*interfaceType)(unsafe.Pointer(t))
		return tt.MethodByName(name)
	}
	ut := t.uncommon()
	if ut == nil {
		return Method{}, false
	}
	// TODO(mdempsky): Binary search.
	for i, p := range ut.exportedMethods() {
		if t.nameOff(p.name).name() == name {
			return t.Method(i), true
		}
	}
	return Method{}, false
}
~~~
`interfaceType` 在上面已经贴出来了
- 在第3行判断是否是 interface 类型，如果是的话，继续调用 interface的 tt.MethodByName，后面继续追踪
- 如果不是interface，则拿到存储的函数地址的slice，遍历这个slice，并跟name的地址获取函数的 name，匹配是否相同

**interfaceType.MethodByName**

~~~
func (t *interfaceType) MethodByName(name string) (m Method, ok bool) {
	if t == nil {
		return
	}
	var p *imethod
	for i := range t.methods {
		p = &t.methods[i]
		if t.nameOff(p.name).name() == name {
			return t.Method(i), true
		}
	}
	return
}

~~~
这个跟上面的逻辑一样，这里有一个`nameOff`的函数，这里是根据地址及偏移量计算出目标地址，也就是进行指针运算

#### TypeOf(err).Method(0)
`TypeOf(err).Method()`跟`MethodByName`一样，如果是interface，则转到 `interfaceType.MethodByName`，如此，直接追踪即可
~~~
func (t *interfaceType) Method(i int) (m Method) {
	if i < 0 || i >= len(t.methods) {
		return
	}
	p := &t.methods[i]
	pname := t.nameOff(p.name)
	m.Name = pname.name()
	if !pname.isExported() {
		m.PkgPath = pname.pkgPath()
		if m.PkgPath == "" {
			m.PkgPath = t.pkgPath.name()
		}
	}
	m.Type = toType(t.typeOff(p.typ))
	m.Index = i
	return
}

~~~
- 直接根据索引，找到对应的`imethod`的结构体
- 然后根据`imethod`结构体中的`nameOff`偏移指针，找到`name`结构体
- 然后调用 `name` 的  `name()` 方法获取名字
- isExported暂时忽略，设计 `name`下`bytes`属性的设计
- 转换m.Type
- 添加m.Index
- 返回
所以，在使用 `MethodByName` 和 `Method` 都是在 `methods` 中查找，不一样的是，`MethodByName`需要遍历 `methods` 这个slice
我们在上面说过， `methods`这里slice是会进行hash排序的，所以**这里面的索引并不是按照函数的声明顺序来排列的！！！**

## 参考文档

- [《Go Interface 源码剖析》](http://legendtkl.com/2017/07/01/golang-interface-implement/)
- [《Go Data Structures: Interfaces》](https://research.swtch.com/interfaces)