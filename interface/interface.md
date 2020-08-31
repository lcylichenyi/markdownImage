







#### 0 interface是什么

> In Object-oriented programming, a **protocol** or **interface** is a common means for unrelated Object (computer science) to communicate with each other. These are definitions of Method (computer programming) and values which the objects agree upon in order to co-operate. ——wiki

在wiki中是这样定义的，interface和protocol类似，都是双方为了交流而作出的约定。

直接拿这套定义去理解golang中的interface可能难以理解，那么可以换种说法，即interface就像包饺子。

具体类型就像饺子馅，而预先定义的接口则就是饺子皮，也就是对具体类型进行了一层封装后，放入桌上（itabTable），随需随拿。最后吃的仍然是馅。（真正使用的仍然是接口中包着的具体类型方法）





#### 1 为什么要使用interface

##### 1.1 **写一个通用的函数**

在golang中不支持泛型，如果不使用interface的话，需要考虑不同类型的入参，复制粘贴n份，累得慌

而有了interface之后，**各种类型都能封装成interface**，因此间接的实现了泛型编程，能够用来写接受多种类型入参的函数。

这一点可以利用来做**单元测试**(mock入参)，以及**各服务之间的解偶**(比如只提供一个redis的增删改查接口，实现层可随意替换实现方式不影响业务)。



##### 1.2 隐藏具体实现

用户只能使用interface提供的方法，而具体的实现细节则不需要暴露。（典型案例context.Context）



##### 1.3 提供插入点

有点像java的静态代理，就是在调用函数的时候，在前面做点别的事情。

举个例子就是在http请求之前加header的实现

```go
type header struct {
    rt  http.RoundTripper
    v   map[string]string
}

func (h header) RoundTrip(r *http.Request) *http.Response {
    for k, v := range h.v {
        r.Header.Set(k,v)
    }
    return h.rt.RoundTrip(r)
}
```





#### 2 interface的结构

interface存在两种interface类型,eface和iface，



##### 2.1 eface

![image-20200825173000226](/Users/lichenyi/Library/Application Support/typora-user-images/image-20200825173000226.png)

eface顾名思义 empty interface，没有定义方法的interface底层结构即为eface。

eface只有_type以及指向数据(拷贝)的指针。



##### 2.2 iface

![image-20200825172901103](/Users/lichenyi/Library/Application Support/typora-user-images/image-20200825172901103.png)

定义了方法的interface底层结构为iface。

iface则还有定义接口方法，因此有有一个tab属性以及指向数据(拷贝)的指针。



```go
type itab struct {
	inter *interfacetype // 接口类型
	_type *_type         // 具体类型
	hash  uint32         // _type的哈希值，在itab下拷贝一份方便使用
	_     [4]byte
	fun   [1]uintptr     //  函数地址表（入口），fun[0] == 0就代表_type没有实现inter
}
```



#### 3 itabTable的设计

![image-20200828011148781](/Users/lichenyi/Library/Application Support/typora-user-images/image-20200828011148781.png)

```go
type itabTableType struct {
   size    uintptr              // entries数组的总大小, 2^n个
   count   uintptr             // 当前数组中实际itab的数量
   entries [itabInitSize]*itab // 哈希表
}
```



##### 3.1 什么是itabTable

**golang有一个itabTable哈希表，即利用空间换时间的思路，存放所有的itab，具体实现方式则通过一个数组(entries)实现**



##### 3.2 如何对itab进行哈希

取itab中的接口类型与实际类型，**分别哈希后取异或**

```go
func itabHashFunc(inter *interfacetype, typ *_type) uintptr {
  // 取itab中的接口类型与实际类型
  // 分别哈希后取异或
   return uintptr(inter.typ.hash ^ typ.hash)
}
```



##### 3.3 itabTable哈希表的**寻址方式**——（二次寻址法）

itabTable作为一个哈希表，插入和读取肯定不可能是每次遍历整个数组，这样非常耗费性能。

因此go中itabTable使用的是quadratic probing

公式为**h(i) = h0 + i*(i+1)/2 mod 2^k**



<img src="/Users/lichenyi/go/src/lcy/markdownImage/interface/image-20200828011937509.png" alt="image-20200828011937509" style="zoom:50%;" />



**h(i)**：**目标位置**

**h0**：起点，也就是一开始**将itab哈希之后的值**。（在这里对itab哈希的实现是通过将interfacetype和itab分别哈希之后异或获得）

**i*(i+1)/2**：**用于防止哈希冲突**，其实就是趋向于1+2+3+4+5+6...的函数表达式，在实现中是一个for循环，不断增加偏移量，比如一开始a+1，如果bucket已被占位或不是目标内容，则下一次找a+1+2的位置，还不是就a+1+2+3。为了防止一直递增超过哈希表（数组）的大小，所以加一个mod 2^k(mod数组的长度)



其实也就是一种**二次寻址法的实现**。



#### 4 itabTable的增与删

##### 4.1 itab的初始化——init方法

**itab需要初始化之后才能插入itabTable**

**遍历接口类型与具体类型**比较具体类型是否实现了所有接口类型， 理论上时间复杂度为O(n^2)

但是实际上因为接口类型与具体类型的插入都是按照字典序排序的，因此实际上**时间复杂度为O(mn)**，使用双指针遍历即可。



```go
func (m *itab) init() string {
	inter := m.inter  // 接口类型
	typ := m._type    // 具体类型
	x := typ.uncommon()

	ni := len(inter.mhdr) // 接口类型的方法数量
	nt := int(x.mcount)   // 具体类型的方法数量
	xmhdr := (*[1 << 16]method)(add(unsafe.Pointer(x), uintptr(x.moff)))[:nt:nt] // 指向具体类型的方法数组
	j := 0
  
  // 遍历每一个接口类型所定义的方法，看实际类型的方法数组中是否有实现，如果没有实现的话就返回没有实现的函数名
imethods:
	for k := 0; k < ni; k++ {
		i := &inter.mhdr[k]
		itype := inter.typ.typeOff(i.ityp)
		name := inter.typ.nameOff(i.name)
		iname := name.name()
		ipkg := name.pkgPath()
		if ipkg == "" {
			ipkg = inter.pkgpath.name()
		}
		for ; j < nt; j++ {
			t := &xmhdr[j]
			tname := typ.nameOff(t.name)
			if typ.typeOff(t.mtyp) == itype && tname.name() == iname {
				pkgPath := tname.pkgPath()
				if pkgPath == "" {
					pkgPath = typ.nameOff(x.pkgpath).name()
				}
				if tname.isExported() || pkgPath == ipkg {
					if m != nil {
						ifn := typ.textOff(t.ifn)
						*(*unsafe.Pointer)(add(unsafe.Pointer(&m.fun[0]), uintptr(k)*sys.PtrSize)) = ifn
					}
					continue imethods
				}
			}
		}
		// 没有找到实现方法的话就把m.fun[0]变成0
		m.fun[0] = 0
		return iname
	}
	m.hash = typ.hash
  // 如果都实现了就返回空字符串
	return ""
}
```





##### 4.2 插入itab至itabTable——itabAdd方法

itab使用接口类型(interfaceType)以及具体类型(_type)初始化之后，就能将itab放置与itabTable。

使用的插入方法正是之前的**二次寻址法**

```go
func itabAdd(m *itab) {
   if getg().m.mallocing != 0 {
      throw("malloc deadlock")
   }

   t := itabTable
   if t.count >= 3*(t.size/4) { // 当itabTable使用率大于75%时就要扩容了
      t2 := (*itabTableType)(mallocgc((2+2*t.size)*sys.PtrSize, nil, true))
      t2.size = t.size * 2
      iterate_itabs(t2.add)
      if t2.count != t.count {
         throw("mismatched count during itab table copy")
      }
      atomicstorep(unsafe.Pointer(&itabTable), unsafe.Pointer(t2))
      t = itabTable
   }
  // 否则直接加
   t.add(m)
}
```

```go
func (t *itabTableType) add(m *itab) {

	mask := t.size - 1
	h := itabHashFunc(m.inter, m._type) & mask // 哈希取余
	for i := uintptr(1); ; i++ {
		p := (**itab)(add(unsafe.Pointer(&t.entries), h*sys.PtrSize)) // 取位置
		m2 := *p  // 取值
		if m2 == m { // 如果已存在，则返回
			return
		}
		if m2 == nil { // 如果找到空位置就插入
			atomic.StorepNoWB(unsafe.Pointer(p), unsafe.Pointer(m))
			t.count++
			return
		}
		h += i // 加上步长,每一次循环时的步长趋近于公式 i*(i+1)/2 
		h &= mask // 取模
	}
}
```







##### 4.3 在itabTable中寻找itab——find方法

在itabTable中根据接口类型以及具体类型寻找itab

使用的搜索模式也是**二次寻址法**

```go
func (t *itabTableType) find(inter *interfacetype, typ *_type) *itab {
   mask := t.size - 1
   h := itabHashFunc(inter, typ) & mask  // 
   for i := uintptr(1); ; i++ {
      p := (**itab)(add(unsafe.Pointer(&t.entries), h*sys.PtrSize))  // 取位置
    
      m := (*itab)(atomic.Loadp(unsafe.Pointer(p)))  // 取值
      if m == nil { // 如果找到空位，说明不存在，结束
         return nil
      }
      if m.inter == inter && m._type == typ {  // 如果找到则返回
         return m
      }
      h += i       // 增加步长
      h &= mask   // 取模
   }
}
```





#### 5 汇编验证

代码：

```go
package main

import "fmt"

func main() {
	var boy Boy
	var person Person
	var superman SuperMan
	superman = boy  // 1. 具体类型包装成接口   第9行
	person = superman // 2. 接口转换   第10行
	fmt.Println(person)

}

type Person interface {
	shout() int64
}

type SuperMan interface {
	shout() int64
	eat()
}

type Boy struct {
	Name int64
}

func (p Boy) shout() int64 {
	return 2333
}

func (p Boy) eat() {

}
```



汇编代码：

```assembly
lichenyi@lichenyideMacBook-Pro test % go tool compile -l -N -S test2.go
"".main STEXT size=356 args=0x0 locals=0xb0
	0x0000 00000 (test2.go:5)	TEXT	"".main(SB), ABIInternal, $176-0
	0x0000 00000 (test2.go:5)	MOVQ	(TLS), CX
	0x0009 00009 (test2.go:5)	LEAQ	-48(SP), AX
	0x000e 00014 (test2.go:5)	CMPQ	AX, 16(CX)
	0x0012 00018 (test2.go:5)	JLS	346
	0x0018 00024 (test2.go:5)	SUBQ	$176, SP
	0x001f 00031 (test2.go:5)	MOVQ	BP, 168(SP)
	0x0027 00039 (test2.go:5)	LEAQ	168(SP), BP
	0x002f 00047 (test2.go:5)	FUNCDATA	$0, gclocals·f14a5bc6d08bc46424827f54d2e3f8ed(SB)
	0x002f 00047 (test2.go:5)	FUNCDATA	$1, gclocals·1a6fe1d1d16fa1a31a2f14d3d1e3cfe1(SB)
	0x002f 00047 (test2.go:5)	FUNCDATA	$3, gclocals·f6aec3988379d2bd21c69c093370a150(SB)
	0x002f 00047 (test2.go:5)	FUNCDATA	$4, "".main.stkobj(SB)
	0x002f 00047 (test2.go:6)	PCDATA	$2, $0
	0x002f 00047 (test2.go:6)	PCDATA	$0, $0
	0x002f 00047 (test2.go:6)	MOVQ	$0, "".boy+48(SP)
	0x0038 00056 (test2.go:7)	XORPS	X0, X0
	0x003b 00059 (test2.go:7)	MOVUPS	X0, "".person+96(SP)
	0x0040 00064 (test2.go:8)	XORPS	X0, X0
	0x0043 00067 (test2.go:8)	MOVUPS	X0, "".superman+80(SP)
	0x0048 00072 (test2.go:9)	MOVQ	"".boy+48(SP), AX        # 将name字段（int64）放入ax寄存器
	0x004d 00077 (test2.go:9)	MOVQ	AX, (SP) 							   # 将name字段放入SP    （两步结合就是将Boy类型放入SP+0位）
	0x0051 00081 (test2.go:9)	CALL	runtime.convT64(SB)      # 智能分析出只有一个int64字段，使用方法convT64分配8字节内存即可（此处有多种方法）
	0x0056 00086 (test2.go:9)	PCDATA	$2, $1
	0x0056 00086 (test2.go:9)	MOVQ	8(SP), AX
	0x005b 00091 (test2.go:9)	MOVQ	AX, ""..autotmp_4+72(SP)
	0x0060 00096 (test2.go:9)	PCDATA	$2, $2
	0x0060 00096 (test2.go:9)	PCDATA	$0, $1
	0x0060 00096 (test2.go:9)	LEAQ	go.itab."".Boy,"".SuperMan(SB), CX   # 将根据具体类型Boy以及接口类型SuperMan生成的itab放入CX寄存器，
	0x0067 00103 (test2.go:9)	PCDATA	$2, $1
	0x0067 00103 (test2.go:9)	MOVQ	CX, "".superman+80(SP)        # 将itab放入superman(类型为iface)的tab字段
	0x006c 00108 (test2.go:9)	PCDATA	$2, $0  
	0x006c 00108 (test2.go:9)	MOVQ	AX, "".superman+88(SP)      # 将数据拷贝到superman(类型为iface)的data字段，至次完成superman接口的赋值
	0x0071 00113 (test2.go:10)	PCDATA	$2, $1
	0x0071 00113 (test2.go:10)	LEAQ	type."".Person(SB), AX  # 将Person接口类型放入ax寄存器
	0x0078 00120 (test2.go:10)	PCDATA	$2, $0
	0x0078 00120 (test2.go:10)	MOVQ	AX, (SP)     # 将ax移到SP+0位 (即将Person接口置于第一个入参)
	0x007c 00124 (test2.go:10)	PCDATA	$2, $1
	0x007c 00124 (test2.go:10)	MOVQ	"".superman+88(SP), AX
	0x0081 00129 (test2.go:10)	PCDATA	$0, $0
	0x0081 00129 (test2.go:10)	MOVQ	"".superman+80(SP), CX
	0x0086 00134 (test2.go:10)	MOVQ	CX, 8(SP)   # 将superman的tab字段放入第二个入参
	0x008b 00139 (test2.go:10)	PCDATA	$2, $0
	0x008b 00139 (test2.go:10)	MOVQ	AX, 16(SP)  # 将superman的data字段放入第三个入参（组合成第二个iface入参）
	0x0090 00144 (test2.go:10)	CALL	runtime.convI2I(SB)   # convI2I方法，convI2I(inter *interfacetype, i iface) (r iface)                                            
	0x0095 00149 (test2.go:10)	PCDATA	$2, $1
	0x0095 00149 (test2.go:10)	MOVQ	32(SP), AX
	0x009a 00154 (test2.go:10)	MOVQ	24(SP), CX
	0x009f 00159 (test2.go:10)	MOVQ	CX, "".person+96(SP)    
	0x00a4 00164 (test2.go:10)	MOVQ	AX, "".person+104(SP)   # 将tab字段与data字段分别给（类型也为iface）person，完成person的赋值
	0x00a9 00169 (test2.go:11)	PCDATA	$0, $2
	0x00a9 00169 (test2.go:11)	MOVQ	CX, ""..autotmp_5+112(SP)
	0x00ae 00174 (test2.go:11)	PCDATA	$2, $0
	0x00ae 00174 (test2.go:11)	MOVQ	AX, ""..autotmp_5+120(SP)
	0x00b3 00179 (test2.go:11)	PCDATA	$0, $3
	0x00b3 00179 (test2.go:11)	MOVQ	CX, ""..autotmp_6+64(SP)
	0x00b8 00184 (test2.go:11)	CMPQ	""..autotmp_6+64(SP), $0
	0x00be 00190 (test2.go:11)	JNE	197
	0x00c0 00192 (test2.go:11)	JMP	341
	0x00c5 00197 (test2.go:11)	PCDATA	$0, $2
	0x00c5 00197 (test2.go:11)	TESTB	AL, (CX)
	0x00c7 00199 (test2.go:11)	PCDATA	$2, $1
	0x00c7 00199 (test2.go:11)	MOVQ	8(CX), AX
	0x00cb 00203 (test2.go:11)	PCDATA	$2, $0
	0x00cb 00203 (test2.go:11)	PCDATA	$0, $3
	0x00cb 00203 (test2.go:11)	MOVQ	AX, ""..autotmp_6+64(SP)
	0x00d0 00208 (test2.go:11)	JMP	210
	0x00d2 00210 (test2.go:11)	PCDATA	$0, $4
	0x00d2 00210 (test2.go:11)	XORPS	X0, X0
	0x00d5 00213 (test2.go:11)	MOVUPS	X0, ""..autotmp_3+128(SP)
	0x00dd 00221 (test2.go:11)	PCDATA	$2, $1
	0x00dd 00221 (test2.go:11)	PCDATA	$0, $3
	0x00dd 00221 (test2.go:11)	LEAQ	""..autotmp_3+128(SP), AX
	0x00e5 00229 (test2.go:11)	MOVQ	AX, ""..autotmp_8+56(SP)
	0x00ea 00234 (test2.go:11)	TESTB	AL, (AX)
	0x00ec 00236 (test2.go:11)	PCDATA	$2, $2
	0x00ec 00236 (test2.go:11)	PCDATA	$0, $5
	0x00ec 00236 (test2.go:11)	MOVQ	""..autotmp_5+120(SP), CX
	0x00f1 00241 (test2.go:11)	PCDATA	$2, $3
	0x00f1 00241 (test2.go:11)	PCDATA	$0, $0
	0x00f1 00241 (test2.go:11)	MOVQ	""..autotmp_6+64(SP), DX
	0x00f6 00246 (test2.go:11)	PCDATA	$2, $2
	0x00f6 00246 (test2.go:11)	MOVQ	DX, ""..autotmp_3+128(SP)
	0x00fe 00254 (test2.go:11)	PCDATA	$2, $1
	0x00fe 00254 (test2.go:11)	MOVQ	CX, ""..autotmp_3+136(SP)
	0x0106 00262 (test2.go:11)	TESTB	AL, (AX)
	0x0108 00264 (test2.go:11)	JMP	266
	0x010a 00266 (test2.go:11)	MOVQ	AX, ""..autotmp_7+144(SP)
	0x0112 00274 (test2.go:11)	MOVQ	$1, ""..autotmp_7+152(SP)
	0x011e 00286 (test2.go:11)	MOVQ	$1, ""..autotmp_7+160(SP)
	0x012a 00298 (test2.go:11)	PCDATA	$2, $0
	0x012a 00298 (test2.go:11)	MOVQ	AX, (SP)
	0x012e 00302 (test2.go:11)	MOVQ	$1, 8(SP)
	0x0137 00311 (test2.go:11)	MOVQ	$1, 16(SP)
	0x0140 00320 (test2.go:11)	CALL	fmt.Println(SB)
	0x0145 00325 (test2.go:13)	MOVQ	168(SP), BP
	0x014d 00333 (test2.go:13)	ADDQ	$176, SP
	0x0154 00340 (test2.go:13)	RET
	0x0155 00341 (test2.go:11)	PCDATA	$2, $-2
	0x0155 00341 (test2.go:11)	PCDATA	$0, $-2
	0x0155 00341 (test2.go:11)	JMP	210
	0x015a 00346 (test2.go:11)	NOP
	0x015a 00346 (test2.go:5)	PCDATA	$0, $-1
	0x015a 00346 (test2.go:5)	PCDATA	$2, $-1
	0x015a 00346 (test2.go:5)	CALL	runtime.morestack_noctxt(SB)
	0x015f 00351 (test2.go:5)	JMP	0
	0x0000 65 48 8b 0c 25 00 00 00 00 48 8d 44 24 d0 48 3b  eH..%....H.D$.H;
	0x0010 41 10 0f 86 42 01 00 00 48 81 ec b0 00 00 00 48  A...B...H......H
	0x0020 89 ac 24 a8 00 00 00 48 8d ac 24 a8 00 00 00 48  ..$....H..$....H
	0x0030 c7 44 24 30 00 00 00 00 0f 57 c0 0f 11 44 24 60  .D$0.....W...D$`
	0x0040 0f 57 c0 0f 11 44 24 50 48 8b 44 24 30 48 89 04  .W...D$PH.D$0H..
	0x0050 24 e8 00 00 00 00 48 8b 44 24 08 48 89 44 24 48  $.....H.D$.H.D$H
	0x0060 48 8d 0d 00 00 00 00 48 89 4c 24 50 48 89 44 24  H......H.L$PH.D$
	0x0070 58 48 8d 05 00 00 00 00 48 89 04 24 48 8b 44 24  XH......H..$H.D$
	0x0080 58 48 8b 4c 24 50 48 89 4c 24 08 48 89 44 24 10  XH.L$PH.L$.H.D$.
	0x0090 e8 00 00 00 00 48 8b 44 24 20 48 8b 4c 24 18 48  .....H.D$ H.L$.H
	0x00a0 89 4c 24 60 48 89 44 24 68 48 89 4c 24 70 48 89  .L$`H.D$hH.L$pH.
	0x00b0 44 24 78 48 89 4c 24 40 48 83 7c 24 40 00 75 05  D$xH.L$@H.|$@.u.
	0x00c0 e9 90 00 00 00 84 01 48 8b 41 08 48 89 44 24 40  .......H.A.H.D$@
	0x00d0 eb 00 0f 57 c0 0f 11 84 24 80 00 00 00 48 8d 84  ...W....$....H..
	0x00e0 24 80 00 00 00 48 89 44 24 38 84 00 48 8b 4c 24  $....H.D$8..H.L$
	0x00f0 78 48 8b 54 24 40 48 89 94 24 80 00 00 00 48 89  xH.T$@H..$....H.
	0x0100 8c 24 88 00 00 00 84 00 eb 00 48 89 84 24 90 00  .$........H..$..
	0x0110 00 00 48 c7 84 24 98 00 00 00 01 00 00 00 48 c7  ..H..$........H.
	0x0120 84 24 a0 00 00 00 01 00 00 00 48 89 04 24 48 c7  .$........H..$H.
	0x0130 44 24 08 01 00 00 00 48 c7 44 24 10 01 00 00 00  D$.....H.D$.....
	0x0140 e8 00 00 00 00 48 8b ac 24 a8 00 00 00 48 81 c4  .....H..$....H..
	0x0150 b0 00 00 00 c3 e9 78 ff ff ff e8 00 00 00 00 e9  ......x.........
	0x0160 9c fe ff ff                                      ....
	rel 5+4 t=16 TLS+0
	rel 82+4 t=8 runtime.convT64+0
	rel 99+4 t=15 go.itab."".Boy,"".SuperMan+0
	rel 116+4 t=15 type."".Person+0
	rel 145+4 t=8 runtime.convI2I+0
	rel 321+4 t=8 fmt.Println+0
	rel 347+4 t=8 runtime.morestack_noctxt+0

```