

<img src="/Users/lichenyi/Documents/plan9_from_bell.jpg" alt="plan9_from_bell" style="zoom:30%;" />



### 前言：

问：什么是plan9？

答：plan9是一个很强的操作系统，但我们只需要学习它的汇编语法。

问：为什么说golang开发者需要学习plan9汇编？

答：因为golang的开发团队和bell实验室（开发了Unix的那个实验室）开发plan9操作系统的钢铁糙汉子开发团队是同一批人，他们非要用，咱也没办法。

问：反编译之后玩Intel和AT&T不香吗？

答：确实可以跳过plan9汇编（比如直接拿机器码反编译出intel汇编来看），但是会让阅读变得非常困难。并且在golang的基础方法中，使用了大量plan9汇编，其中包含了一些如4个伪寄存器等plan9特有的语法，能让人更容易读懂代码，少绕弯路。（再说汇编都大同小异，秒学完好伐！）

问：学plan9汇编有什么好处？

答：学习plan9能让你在golang开发者中脱颖而出，随时随地掏出大汇编对bug进行降维打击，成为同事们心中的偶像，**获得plmm以及sqgg的芳心**，从此走上人生巅峰。







### 一.plan9简介

1.Plan-9是一款神奇的新版Unix，几乎是由70年代当初开发Unix系统的同一个团队开发的。

2.目的就是要最终解决Unix最初的诺言：一切皆为文件（先进的9P虚拟文件系统协议最终让所有东西都成为了文件。目录变成了“命名空间”，资源被映射成了文件。）

> （你可以通过对/proc目录(现在应该成其为一个命名空间)里的一个文件使用“cat”命令来查看进程的情况。同样，打开一个网络连接的方式变成了打开/net/tcp目录里的一个文件。”iotcl”系统调用在这个系统里完全被根除了，因为基于操作系统上的现代文件形式中的这种怪胎已经不再需要了。）

3.Plan-9实际上没有解决任何问题，并且开发者们不屑于商业化，暂时不打算与Unix兼容。这也是为什么plan9操作系统按理说比Unix强但是却没有推广起来的原因。



### 二.plan9语法的一些特点

1.没有 push 和 pop,栈的调整是通过对硬件 SP 寄存器进行运算来实现的

2.常数在 plan9 汇编用 $num 表示，可以为负数，默认情况下为十进制。

3.操作数方向与intel相反，与AT&T类似

```asm
SUBQ	$24, SP  // 对 SP 做减法，为函数分配函数栈24字节大小的帧 (因为栈是从高地址向低地址增长的)
...
中间的一堆代码
...
ADDQ	$24, SP  // 对 SP 做加法，清除函数栈帧
```

4.数据搬运的长度由 MOV 的后缀决定

```assembly
// plan9
MOVB $1, DI      // 1 byte
MOVW $0x10, BX   // 2 bytes
MOVD $1, DX      // 4 bytes
MOVQ $-10, AX     // 8 bytes

// intel
mov rax, 0x1   // 8 bytes
mov eax, 0x100 // 4 bytes
mov ax, 0x22   // 2 bytes
mov ah, 0x33   // 1 byte
mov al, 0x44   // 1 byte
```

5.为了简化汇编代码的编写,引入了4个伪寄存器。(其实就是Go汇编语言对CPU的重新抽象)

- `FP`: Frame pointer: arguments and locals.
- `PC`: Program counter: jumps and branches.
- `SB`: Static base pointer: global symbols.
- `SP`: Stack pointer: top of stack.

四个伪寄存器和X86/AMD64的内存和寄存器的相互关系如下图：

![四个伪寄存器](/Users/lichenyi/Documents/四个伪寄存器.png)

在AMD64环境，伪PC寄存器其实是IP指令计数器寄存器的别名。伪FP寄存器对应的是函数的帧指针，用来访问函数的参数和返回值。伪SP栈指针对应的是当前函数栈帧的底部（不包括参数和返回值部分），用于定位局部变量。伪SP是一个比较特殊的寄存器，因为还存在一个同名的SP真寄存器。真SP寄存器对应的是栈的顶部，用于定位调用其它函数的参数和返回值。

当需要区分伪寄存器和真寄存器的时候只需要记住一点：伪寄存器需要一个标识符和偏移量为前缀，如果没有标识符前缀则是真寄存器。比如`(SP)`、`+8(SP)`没有标识符前缀为真SP寄存器，而`a(SP)`、`b+8(SP)`有标识符为前缀表示伪寄存器。



6.**被调用函数的入参与出参都在调用函数的栈帧中**。

![栈示意图](/Users/lichenyi/Downloads/栈示意图.png)

在这一点和c语言有一点不一样，c当入参小于6个时会使用寄存器，出参也只允许有一个，想要有多返回值要么就是返回一个指针，要么就是把入参当出参用。而golang则一律使用栈来传输入参与出参，所以函数调用有**一定的性能损耗**（会比c慢一点）。Go编译器是通过**函数内联**来缓解这个问题的影响



PS:

在这里提一嘴，golang可以通过命令查看build过程中究竟干了些什么

```shell
go build -n filename.go
```



build分为三个阶段，compile, link 以及buildId。

> buildId:Buildid displays or updates the build ID stored in a Go package or binary.

而在compile过程中做了下图这六件事，大致就是

- 词法分析：根据空格等符号分词
- 语法分析：生成AST
- 语义分析：类型检查+逃逸分析+内联等  （**禁止函数内联就是操作这个步骤**）
- 中间码生成：替换一些底层函数（如判断使用makeslice64或makeslice）
- 代码优化：顾名思义，就是搞提升并行，指令优化，利用寄存器等代码优化
- 机器代码生成：根据GOARCH，生成plan9

![编译器原理](/Users/lichenyi/Documents/编译器原理.png)





### 三.plan9的函数声明

```assembly
// func add(a, b int) int
//   => 该声明定义在同一个 package 下的任意 .go 文件中
//   => 只有函数头，没有实现
TEXT pkgname·add(SB), NOSPLIT, $0-8
    MOVQ a+0(FP), AX
    MOVQ a+8(FP), BX
    ADDQ AX, BX
    MOVQ BX, ret+16(FP)
    RET
```



```go
                              参数及返回值大小
                                  | 
 TEXT pkgname·add(SB),NOSPLIT,$32-32
       |        |               |
      包名     函数名         栈帧大小(局部变量+可能需要的额外调用函数的参数空间的总大小，但不包括调用其它函数时的 ret address 的大小)
```



PS: golang会自动为每个函数加入一段栈扩容检测的代码，而对于小函数会进行优化，不加入栈扩容检测。而NOSPLIT也能强制定义取消栈扩容检查，好处则是速度可以一定程度变快，缺点则也很明显，空间不足就GG了。



找个例子试一下：

>sync/atomic/doc.go 中定义了CompareAndSwapInt32方法
>
>同级目录下有asm.s ，可以看到对应方法的汇编代码。

```assembly
TEXT ·CompareAndSwapUint32(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Cas(SB)
```

>然后可以跟踪到runtime/internal/asm_amd64.s 中

```assembly
// bool Cas(int32 *val, int32 old, int32 new)
// Atomically:
//	if(*val == old){
//		*val = new;
//		return 1;
//	} else
//		return 0;
TEXT runtime∕internal∕atomic·Cas(SB),NOSPLIT,$0-17
	MOVQ	ptr+0(FP), BX   ; 第一个参数命名为addr，放入BP(MOVQ，完成8个字节的复制)
	MOVL	old+8(FP), AX   ; 第二个参数命名为old，放入AX
	MOVL	new+12(FP), CX  ; 第三个参数命名为new，放入CX
	LOCK                  ; 锁内存总线操作，防止其它CPU干扰
	CMPXCHGL	CX, 0(BX)   ; CMPXCHGL，该指令会把AX中的内容和第二个操作数中的内容比较，如果相等，那么把第一个操作数内容赋值给第二个操作数，换言之则是将old与addr中的内容做比较，如果相等，则新值覆盖旧值。
	SETEQ	ret+16(FP)  
	RET
```

通过追溯源代码可以进一步确认golang中atomic包中的方法是通过单指令防止因cpu调度等原因被中断，从而解决临界区问题。所以至此可以断言atomic方法没有使用信号量，因此也没有内核态向用户态的转变这一消耗，是高性能的实现并发安全的方式



### 四.解决实际问题

除了直接阅读源代码中的汇编代码之外，还可以将go代码进行编译，从而得到编译后的代码（这时候就不含伪寄存器了）

命令：

-l: 禁止内联

-N: 禁止优化

-S: 输出到标准输出

```shell
go tool compile -S -N -l main.go
```



main.go

```go
package main

func main() {
	_ = add(3,5)
}

func add(a, b int) int {
	return a+b
}
```

编译后：

```assembly
"".main STEXT size=68 args=0x0 locals=0x20
	0x0000 00000 (main.go:3)	TEXT	"".main(SB), ABIInternal, $32-0 ; BP 8个字节。2个入参+1个出参 24个字节，所以32个字节
	0x0000 00000 (main.go:3)	MOVQ	(TLS), CX    ; 加载g结构体指针,可以查看runtime的getg()方法获取的*g结构
	0x0009 00009 (main.go:3)	CMPQ	SP, 16(CX) ;SP栈指针和g结构体中stackguard0成员比较 判断是否扩容
	0x000d 00013 (main.go:3)	JLS	61 ; 需要扩容就跳过去 （以上部分在nosplit模式以及小函数中没有）
	0x000f 00015 (main.go:3)	SUBQ	$32, SP  ; 栈扩容32个字节
	0x0013 00019 (main.go:3)	MOVQ	BP, 24(SP) ; 将bp寄存器中的值存入(物理寄存器)SP偏移24字节的8个字节位
	0x0018 00024 (main.go:3)	LEAQ	24(SP), BP ; 将24(SP)的地址置入BP（其实就是为了交给子函数，用来找回它的父函数）
	0x001d 00029 (main.go:3)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (main.go:3)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (main.go:3)	FUNCDATA	$3, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x001d 00029 (main.go:4)	PCDATA	$2, $0
	0x001d 00029 (main.go:4)	PCDATA	$0, $0 ; funcdata与pcdata与GC有关，可以忽略
	0x001d 00029 (main.go:4)	MOVQ	$3, (SP) ; 赋值3
	0x0025 00037 (main.go:4)	MOVQ	$5, 8(SP)  ; 赋值5
	0x002e 00046 (main.go:4)	CALL	"".add(SB)  ; 调用add函数，这时候16(SP)的位置已经空出来用于放返回值
	0x0033 00051 (main.go:5)	MOVQ	24(SP), BP
	0x0038 00056 (main.go:5)	ADDQ	$32, SP ; 缩栈
	0x003c 00060 (main.go:5)	RET ; 结束
	0x003d 00061 (main.go:5)	NOP
	0x003d 00061 (main.go:3)	PCDATA	$0, $-1
	0x003d 00061 (main.go:3)	PCDATA	$2, $-1
	0x003d 00061 (main.go:3)	CALL	runtime.morestack_noctxt(SB)
	0x0042 00066 (main.go:3)	JMP	0
	0x0000 65 48 8b 0c 25 00 00 00 00 48 3b 61 10 76 2e 48  eH..%....H;a.v.H
	0x0010 83 ec 20 48 89 6c 24 18 48 8d 6c 24 18 48 c7 04  .. H.l$.H.l$.H..
	0x0020 24 03 00 00 00 48 c7 44 24 08 05 00 00 00 e8 00  $....H.D$.......
	0x0030 00 00 00 48 8b 6c 24 18 48 83 c4 20 c3 e8 00 00  ...H.l$.H.. ....
	0x0040 00 00 eb bc                                      ....
	rel 5+4 t=16 TLS+0
	rel 47+4 t=8 "".add+0
	rel 62+4 t=8 runtime.morestack_noctxt+0
"".add STEXT nosplit size=25 args=0x18 locals=0x0
	0x0000 00000 (main.go:7)	TEXT	"".add(SB), NOSPLIT|ABIInternal, $0-24  ; 因为入参与出参由调用函数提供，所以栈桢为0，出入参总和24个字节
	0x0000 00000 (main.go:7)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (main.go:7)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (main.go:7)	FUNCDATA	$3, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	0x0000 00000 (main.go:7)	PCDATA	$2, $0
	0x0000 00000 (main.go:7)	PCDATA	$0, $0
	0x0000 00000 (main.go:7)	MOVQ	$0, "".~r2+24(SP)
	0x0009 00009 (main.go:8)	MOVQ	"".a+8(SP), AX
	0x000e 00014 (main.go:8)	ADDQ	"".b+16(SP), AX
	0x0013 00019 (main.go:8)	MOVQ	AX, "".~r2+24(SP)
	0x0018 00024 (main.go:8)	RET
	0x0000 48 c7 44 24 18 00 00 00 00 48 8b 44 24 08 48 03  H.D$.....H.D$.H.
	0x0010 44 24 10 48 89 44 24 18 c3                       D$.H.D$..
```





查看slice作为参数的情况，可以发现把一个slice作为参数实际上是传了3个参数，地址指针+len+cap，这一点可以通过代码中sliceHeader结构体得到进一步证实。

ps：golang中字符串有16个字节，也是地址指针+len,结构体为stringHeader。所以无法进行字符串修改。有一个比较有意思的设计点在于因为stringHeader前两个部分与sliceHeader相同，因为plan9汇编是没有类型的，大家都是一块内存，所以可以直接由slice转化为string。

```go
package main

func main() {
	s := make([]int, 3, 10)
	_ = f(s)
}

func f(s []int) int {
	return s[1]
}
```

![slice图片](/Users/lichenyi/Documents/slice图片.png)







至于想用汇编进行逃逸分析的人，个人认为是没必要的。直接gcflags即可。

以下代码可供玩一下，一个是不逃逸，一个是逃逸的。逃逸到堆上一般会造成GC压力，但是另一方面也节省了栈的空间。

```go
package main

import ()

func foo() *int {
    var x int
    return &x
}

func bar() int {
    x := new(int)
    *x = 1
    return *x
}

func main() {}
```



可以直接

```shell
go run -gcflags '-m -l' main.go 
```







### 五.栈扩容

![栈扩容](/Users/lichenyi/Documents/栈扩容.png)



- stack.lo: 栈空间的低地址
- stack.hi: 栈空间的高地址
- stackguard0: stack.lo + StackGuard, 用于stack overlow的检测
- StackGuard: 保护区大小，常量Linux上为880字节
- StackSmall: 常量大小为128字节，用于小函数调用的优化



栈扩容检测有时候也会引入一定的问题，比如某厂在大量全双工PUSH中使用GPRC的时候导致所有栈的大小翻一倍，以至于出现线上事故。也是值得警惕的。



### 六.Go 语言的编译指示

在编写go函数时也可以diy一些编译行为，个人认为只有`//go:nosplit`以及`//go:noinline`有点用，其他都没啥实际作用。

> https://segmentfault.com/a/1190000016743220









