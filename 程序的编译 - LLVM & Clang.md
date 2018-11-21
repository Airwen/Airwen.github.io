# 程序的编译 - LLVM & Clang

本篇是对：

[编译器](https://objccn.io/issue-6-2/)
[Mach-O 可执行文件](https://objccn.io/issue-6-3/)
《深入理解计算机系统（原书第3版）》—— Randal E. Bryant & David R.O'Hallaron 机械工业出版社
孙源关于Clang的分享视频

的综合阅读笔记

## 在《深入理解计算机系统（原书第3版）》这本书中这样描述程序的编译过程的：

C语言程序的生命周期是从高级语言程序开始的，但是它要从“高级语言”转换到“可执行目标文件(由一些列“低级”机器语言指令组成)”这一过程，分为四个阶段：

* 预处理阶段：`#include`命令告诉与处理器读取系统头文件，并把它直接插入程序文本中。

* 编译阶段：以文本格式描述了一条低级机器语言指令。(汇编语言为不同高级语言的不同编译器提供了**通用的输出语言**) —— 生成源文件的汇编代码。

* 汇编阶段：汇编器将“编译阶段”输出的文件翻译成“机器语言指令”，把这些指令打包成一种“可重定位目标文件(relocatable object program)”的文件格式 —— 将汇编代码转换成二进制**目标代码**文件。（目标代码是机器代码的一种形式，它包含所有指令的二进制表示，但是还没有填入全局值的地址）。
机器代码的另一种表现形式是**可执行代码** —— 也就是处理器执行的代码格式。

> 扩展：在Linux系统中，带`-d`命令行标志的程序OBJDUMP(表示“object dunmper”)是一种**反汇编器(disassembler)**  —— 这种程序可以根据机器代码产生一种类似于汇编代码的格式。得到是每个指令的字节序列，以及每条指令值和该指令所占的字节数，以及对应的汇编语言。反汇编器只是基于机器代码文件中的字节序列(就是下面代码实例中中间那串16进制)来确定汇编代码。它不需要访问该程序中源代码会汇编代码。

```
hellowoeld.o:	file format Mach-O 64-bit x86-64

Disassembly of section __TEXT,__text:
_main:
       0:	55 	pushq	%rbp
       1:	48 89 e5 	movq	%rsp, %rbp
       4:	48 83 ec 20 	subq	$32, %rsp
       8:	48 8d 05 25 00 00 00 	leaq	37(%rip), %rax
       f:	c7 45 fc 00 00 00 00 	movl	$0, -4(%rbp)
      16:	89 7d f8 	movl	%edi, -8(%rbp)
      19:	48 89 75 f0 	movq	%rsi, -16(%rbp)
      1d:	48 89 c7 	movq	%rax, %rdi
      20:	b0 00 	movb	$0, %al
      22:	e8 00 00 00 00 	callq	0 <_main+0x27>
      27:	31 c9 	xorl	%ecx, %ecx
      29:	89 45 ec 	movl	%eax, -20(%rbp)
      2c:	89 c8 	movl	%ecx, %eax
      2e:	48 83 c4 20 	addq	$32, %rsp
      32:	5d 	popq	%rbp
      33:	c3 	retq
```

* 链接阶段：将预编译的`#include`某个单独的预编目标文件，以某种方式合并到“汇编阶段”结果文件中。**链接器(ld)**就负责这种合并。然后了一个可执行目标文件，可以加载到内存中，有系统执行。

> 即机器执行的程序只是一个字节序列，这些字节序列是对一系列指令的编码。
> 与机器代码的二进制格式相比，汇编代码主要特点是它用可读性更好的的文本格式表示。
> 链接器的任务之一就是位函数调用找到匹配的函数的可执行带啊嘛的位置


## 那么 iOS 或 OS X 中的一个可执行文件 (也叫做 Mach-O executable) 是如何生成的？

Xcode的默认编译器是`clang`。`clang`的功能首先对`Objective-c`代码做分析，然后将其转换为低级的类汇编代码：LLVM Intermediate Representation(LLVM 中间表达码)。接着 LLVM 会执行相关指令将 LLVM IR 编译成目标平台上的本地字节码，这个过程的完成方式可以是即时编译 (Just-in-time)，或在编译的时候完成。

那么Objective-C的`.m`文件，源文件需要几个不同的阶段，我们可以让通过 clang 命令观察(下面用是我的MyLearnPractice项目的`main.m`文件做例子)：

```
➜  MyLearnPractice clang -ccc-print-phases main.m
0: input, "main.m", objective-c
1: preprocessor, {0}, objective-c-cpp-output
2: compiler, {1}, ir
3: backend, {2}, assembler
4: assembler, {3}, object
5: linker, {4}, image 
6: bind-arch, "x86_64", {5}, image
```

预处理     |   词法解析标记  -> 语法解析  ->  静态分析  |        代码生成和优化          |     汇编器      |  链接器
---------------|---------------------------------------|------------------------------|----------------|--------------
 符号化|1. 将符号化的内容转化为一颗解析树(parse tree) |1. 将`AST`转换为更低级的中间码(LLVM IR)|将汇编代码转换为目标对象文件| 将多个目标对象文件合并为一个可执行文件 (或者一个动态库)
 宏定义展开		 |2. 解析树做语义分析        |2. 对生成的中间码做优化	  |    |    |
 #include的展开|3. 输出一棵抽象语法树（Abstract Syntax Tree* (AST)）|3. 生成特定目标代码 |    |    |
  | |4. 生成特定目标代码 |    |    |
  | |4. 输出汇编代码 |    |    |

> Objective-c的编译过程从预处理到编译中间多出来了很多处理，自己先猜测一下：
> 是根据Objective-c语言特性，做了许多特性处理比如关键字，property，内存管理，以及根据Runtime机制，转化为相应的代码。

### 预处理阶段：

预处理器会处理源文件中的宏定义，将代码中的宏用其对应定义的具体内容进行替换：
    
> 预处理器对`#import <Foundation/Foundation.h>`这行代码的处理是用 Foundation.h 文件中的内容去替换这行代码，如果 Foundation.h 中也使用了类似的宏引入，则会按照同样的处理方式用各个宏对应的真正代码进行逐级替代。

`clang -E xxx.m` 进行宏展开的预处理结果是如下所示：

选项意义：

`-E` - 编译器命令行选项是将源代码处理到预处理阶段的结果。
`-fmodules` - 默认把iOS库打成modules的形式。

### 词法解析标记

每一个 `.m` 源文件里都有一堆的 "声明" 和 "定义" 。这些代码文本都会从 string 转化成特殊的标记流。可以用clang 命令来将xxx.m中代码的标记流导出:
`clang -Xclang -dump-tokens xxx.m`

或

`clang -fmodules -fsyntax-only -Xclang -dump-tokens xxx.m`

选项意义：

`-fsyntax-only` - 仅仅告诉clang仅仅解到语法层面(词法解析不会校验语义)

`-Xclang -dump-tokens` - 透传给整整干活的编译器Fronted —— 把语法解析的Token打印出来。

```
int 'int'	 [StartOfLine]	Loc=<main.m:71:1>
identifier 'main'	 [LeadingSpace]	Loc=<main.m:71:5>
l_paren '('		Loc=<main.m:71:9>
r_paren ')'		Loc=<main.m:71:10>
l_brace '{'	 [LeadingSpace]	Loc=<main.m:71:12>
identifier 'NSLog'	 [StartOfLine] [LeadingSpace]	Loc=<main.m:72:5>
l_paren '('		Loc=<main.m:72:10>
at '@'		Loc=<main.m:72:11>
string_literal '"hello, %@"'		Loc=<main.m:72:12>
comma ','		Loc=<main.m:72:23>
at '@'	 [LeadingSpace]	Loc=<main.m:72:25>
string_literal '"world"'		Loc=<main.m:72:26>
r_paren ')'		Loc=<main.m:72:33>
semi ';'		Loc=<main.m:72:34>
return 'return'	 [StartOfLine] [LeadingSpace]	Loc=<main.m:73:5>
numeric_constant '0'	 [LeadingSpace]	Loc=<main.m:73:12>
semi ';'		Loc=<main.m:73:13>
r_brace '}'	 [StartOfLine]	Loc=<main.m:74:1>
eof ''		Loc=<main.m:94:4>
ld: file not found: /var/folders/dj/9xwhf7vd28zbyl921ft8kf_w0000gn/T/main-719d39.o
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```


### 语法分析(解析)

在Clang中有Parse和Sema两个模块配合完成语义的解析，这部会验证语法是否正确。

接下来要说的东西比较有意思：之前生成的标记流将会被解析成一棵抽象语法树 (abstract syntax tree -- AST)。由于 Objective-C 是一门复杂的语言，因此解析的过程不简单。解析过后，源程序变成了一棵抽象语法树：一棵代表源程序的树。可以通过`clang -Xclang -ast-dump -fsyntax-only xxx.m`查看这一阶段的处理结果是什么样的。

这步的命令长啥样捏？

`clang -fmodules -fsyntax-only -Xclang -ast-dump xxx.m`

`-ast-dump` -  dump ast，输出语法解析生成的抽象语法树(AST)。

在抽象语法树中的每个节点都标注了其对应源码中的位置，同样的，如果产生了什么问题，clang 可以定位到问题所在处的源码位置[clang AST 介绍](http://clang.llvm.org/docs/IntroductionToTheClangAST.html)


### 静态分析(Static Analysis)

一旦编译器把源码生成了抽象语法树，编译器可以对这棵树进行代码静态分析处理 —— Xcode "Run" 下面的 “Analyze”选项，可以找出代码非语法性错误：

它是通过模拟代码执行路径，分析出control-flow graph(CFG)

静态分析阶段，除了类型检查外，还会做许多其它一些分析。如果你把 clang 的代码仓库 clone 到本地，然后进入目录 lib/StaticAnalyzer/Checkers，你会看到所有静态检查内容,[*Unofficial* Automated Mirror of LLVM on Github](https://github.com/llvm-mirror)。
 
> 静态和动态类型检查
> 一般会把类型分为两类：动态的和静态的。动态的在运行时做检查，静态的在编译时做检查。以往，编写代码时可以向任意对象发送任何消息，在运行时，才会检查对象是否能够响应这些消息。由于只是在运行时做此类检查，所以叫做动态类型。
> 至于静态类型，是在编译时做检查。当在代码中使用 ARC 时，编译器在编译期间，会做许多的类型检查：因为编译器需要知道哪个对象该如何使用。例如，如果 myObject 没有 hello 方法，那么就不能写如下这行代码了`[myObject hello]`


###代码生成

IR代码生成，CodeGen负责根据生成的语法树从顶至下遍历，翻译成LLVM IR —— LLVM的一个中间语言。
LLVM IR是(编译器)Frontend的输出，也是LLVM Backend的输入，前后端的桥接语言。

与Objective-C Runtime的桥接：

* Class/Meta Class/ Protocol/ Category 内存结构的生成，并存放到指定section中。
  (例如Class: _DATA, _objc_classrefs , _DATA Segment也会保存声明的static变量)
  
 > `.o`或者`.out`之类的可执行文件都会包含：
 > 若干个Segment:`__PAGEZERO`、`__TEXT`、`__DATA`、`__LINKEDIT`
 > 每个Segment又有若干个Section

* Method/Ivar/Property内存结构生成
* 组成 method_list/ ivar_list/ property_list并填入Class
* Non-Fragile ABI: 为每个Ivar合成OBJC_IVAR_$_偏移值常量
* 存取Ivar的语句(_ivar = 123; int a = _ivar;)转写成base+OBJC_IVAR_$_的形式 —— base：对象基地址。
* 将语法树中的`ObjCMessageExpr`翻译成相应版本的`objc_msgSend`，对`super`关键字的调用翻译成`objc_msgSendSuper`
* 根据修饰 `strong`/ `weak` / `copy` / `atomic` 合成 @property 自定实现的 setter/getter
* 处理 `@synthesize`
* 生成 block_layout 的数据结构
* block变量的 capture (__block / __weak)
* 生成 _block_invoke 函数
* ARC: 分析对象引用关系，将`objc_storeStrong`/`objc_storeWeak`等ARC代码插入。
* 将 `ObjCAutoreleasePoolStmt`(生成语法树中的“节点”)转译成objc_autoreleasePoolPush/Pop
* 实现自动调用`[super deallic]`
* 为每个拥有ivar的Class合成 `.cxx_destrucot`方法来自动释放类的成员变量。代替MRC时代的`self.xxx = nil`

生成LLVM IR代码的命令行：

`clang -S -fobjc-arc -emit-llvm mian.m -o main.ll`

### 优化 

`clang -O3 -S -fobjc-arc -emit-llvm mian.m -o main.ll`

优化选项设置 Target -> Build Setting -> Code Generation -> Optimization Level:

`-O0`、`-O1`、`-O2`、`-O3`、`-Os` 

Xcode 9默认选项是`-Os`。

优化是减少栈的操作：内存分配，寄存器的移动...。苹果官方用了生成斐波那契数列的函数来解释，原来递归的形式调用的，优化之后变成了尾递归的形式。
不会真正的用递归执行，整个过程都是在函数内完成。

> 下一步是要去了解“尾递归”？

### 生成字节码`.bc`

不做优化的编译命令：

`clang -O0 -emit-llvm factorial.c  -c -o factorial.bc && llvm-dis < factorial.bc`

有优化效果的编译命令：

`clang -O3 -emit-llvm factorial.c  -c -o factorial.bc && llvm-dis < factorial.bc`

`clang` 完成代码的标记，解析和分析后，接着就会生成  LLVM 字节码（绝大多数情况下是二进制码格式）。

命令行选项：

`clang`的` -c `命令行选项，会编译并汇编该代码，产生目标代码文件。
`clang`的` -o xxx`命令行选项，产生最终的可执行代码文件xxx。

我们可以执行`clang -O3 -emit-LLVM hello.c -c -o hello.bc`命令把“代码”编译成“LLVM 字节码”，然后通过`llvm-dis < hello.bc | less`命令查看刚刚生成的二进制文件。

### 汇编代码 - 生成Target相关的汇编代码`.s`

我们可以用下面的命令让`clang`输出汇编代码，这有[helloworld.c]()生成了一个简单的汇编码文件[helloworld.s]()：

`xcrun clang -S -o - helloworld.c | open -f`

或

`clang -S -fobjc-arc mian.m -o main.s`

编译器选项：

`-S`: clang 命令行选项 —— 使用`-S`生成Target相关的汇编代码。
`| open -f` 将结果输出到一个临时文件中。

下面用了很大篇幅来解释，汇编代码：

会生成一些以点`.`开头的行都是知道汇编器和链接器工作的“伪指令” —— 相对于**汇编指令**的“伪”？，而其它的则是实际的`x86_64`汇编代码(缩进去的行都对应一条机器指令)。这是我生成的编译的汇编文件，下面取一些关键行解释：

`.p2align	4, 0x90` : 指令指出了后面代码的对齐方式。在我们的代码中，后面的代码会按照 16(2^4) 字节对齐，如果需要的话，用 0x90 补齐。问什么是“代码对齐方式”

`.cfi_startproc`: 指令通常用于函数的开始处。CFI 是调用帧信息 (Call Frame Information) 的缩写。这个调用 帧 以松散的方式对应着一个函数。当开发者使用 debugger 和 step in 或 step out 时，实际上是 stepping in/out 一个调用帧。这个指令给了函数一个`.eh_frame`入口，这个入口包含了一些调用栈的信息（抛出异常时也是用其来展开调用帧堆栈的）。这个指令也会发送一些和具体平台相关的指令给 CFI。它与后面的`.cfi_endproc`相匹配，以此标记出`main()`函数结束的地方。

`pushq	%rbp`第一个汇编代码

> 在 OS X上，我们会有 X86_64 的代码，对于这种架构，有一个东西叫做ABI(应用二进制接口 application binary interface)，ABI 指定了函数调用是如何在汇编代码层面上工作的。

在函数调用期间，ABI 会让**`rbp`寄存器**(基础指针寄存器 base pointer register) 被保护起来。
当函数调用返回时，确保**`rbp`寄存器**的值跟之前一样，这是属于`main`函数的职责。`pushq %rbp`将`rbp`的值push到栈中，以便我们以后将其pop出来。

> 基础指针寄存器 base pointer register 是干啥的？

`movq	%rsp, %rbp`:  将把局部变量放置到栈上 —— `mov`类指令吧数据从原位置复制到目的位置，不做任何变化，它是由四条指令组成：`movb`、`movw`、`movl`、`movq`。

所以，大多数汇编代码指令都有一个字符后缀，表明操作数的大小。x86-64指令集同样包括完整的针对字节、字和双字指令(64位机器中，指针长8字节)：

  C声明   | Intel 数据    | 汇编代码后缀  | 大小(字节)
--------- | -------------| -------------| ----------
  char    |     字节     |      b       |     1     
  shor    |     字       |      w       |     2     
  int     |     双字     |      l       |     4     
  long    |     四字     |      q       |     8     
  char*   |     四节     |      q       |     8     
  float   |    单精度    |      s       |     4     
 double   |    双精度    |      l       |     8     

> 注意：汇编代码也适用猴嘴“l”来表示4字节整数和8字节双精度浮点数。这不会产生歧义，因为浮点数使用的是一组完全不同的指令和寄存器。
> 引用自《深入理解计算机系统》—— Randal E. Bryant & David R.O'Hallaron 机械工业出版社 P119 3.3章节。

`subq	$32, %rsp`: 将栈指针移动 32 个字节，也就是函数会调用的位置。
我们先将老的栈指针存储到`rbp`中，然后将此作为我们局部变量的基址，接着我们更新堆栈指针到我们将会使用的位置。

```
	addq	$32, %rsp
	popq	%rbp
	retq
	.cfi_endproc
```

`addq	$32, %rsp`把堆栈指针`rsp`上移 32 字节。最后，把之前存储至`rbp`中的值从栈中弹出来，然后调用`ret`返回调用者，`ret`会读取出栈的返回地址。 `.cfi_endproc`平衡了`.cfi_startproc`指令。

`%rip`, 在x86-64中使用`%rip`表示**程序计数器**。`程序计数器`则是给出将要执行的下一条指令在内存中的地址。

> 对于C语言程序猿隐藏的处理器状态，对与(x86-64的)机器代码都是可见的：

* **程序计数器**(通常称为“PC”，在x86-64中用`%rip`表示)给出将要执行的下一条指令在内存中的地址。
* 整数**寄存器文件**包含16个*命名位置*，(这16个*命名位置*)分别存储16位的值 —— 16个存储64位值的*通用目的寄存器x*。
这些寄存器可以`存储地址(也就是C语言中的指针)`或整数数据。
有的寄存器被用来存储重要的程序状态，而其他寄存器用来保存临时数据 —— 过程参数和局部变量，以及函数返回值。
* 条件码寄存器保存着最近执行的算数或	或逻辑指令的状态信息。它用来实现控制数据流中的条件变化 —— 例如if或while。
* 一组向量寄存器可以存放一个或多个整数或浮点数。

《深入理解计算机系统》书里的列出了这16个寄存器.它们的名字都是以`%r`来头，不过后面还跟着一些不同的命名规则的名字，这是有指令集历史演化造成的。

 bit index  | 63       | 31       | 15       | 7      0 |   作用     	|
----------- | -------- | -------- | -------- | -------- | ------------- |
            |   %rax   |    %eax  |   %ax    |   %al    |   返回值   	|
            |   %rbx   |    %ebx  |   %bx    |   %bl    |  被调用者保存   |
            |   %rcx   |    %ecx  |   %cx    |   %cl    |   第4个参数    |
            |   %rdx   |    %edx  |   %dx    |   %dl    |   第3个参数    |
            |   %rsi   |    %esi  |   %si    |   %sil   |   第2个参数    |
            |   %rdi   |    %edi  |   %di    |   %dil   |   第1个参数    |
            |   %rbp   |    %ebp  |   %bp    |   %bpl   |  被调用者保存   |
            |   %r8    |    %r8d  |   %r8w   |   %r8b   |   第5个参数    |
            |   %r9    |    %r9d  |   %r9w   |   %r9b   |   第6个参数    |
            |   %r10   |   %r10d  |   %r10w  |   %r10b  |   调用者保存   |
            |   %r11   |   %r11d  |   %r11w  |   %r11b  |   调用者保存   |
            |   %r12   |   %r12d  |   %r12w  |   %r12b  |  被调用者保存   |
            |   %r13   |   %r13d  |   %r13w  |   %r13b  |  被调用者保存   |
            |   %r14   |   %r14d  |   %r14w  |   %r14b  |  被调用者保存   |
            |   %r15   |   %r15d  |   %r15w  |   %r15b  |  被调用者保存   |

> 整数寄存器，所有16个寄存器的低位部分都可以作为字节、字（16bit）、双字（32bit）、四字（64bit）数字来访问
> 引用自《深入理解计算机系统》—— Randal E. Bryant & David R.O'Hallaron 机械工业出版社 P119～120 3.4章节。

### 汇编器 - 生成Target相关Object(Mach-O) 

`clang -S -fobjc-arc -emit-llvm mian.m -o main.o`

命令行选项：

`-o`命令行选项改成 —— output的文件类型改成`.o`(Target相关Object(Mach-O))

汇编器将可读的汇编代码转换为机器代码。它会创建一个目标对象文件，一般简称为*对象文件*。这些文件以`.o`结尾。
如果用`Xcode`构建应用程序，可以在工程的 *derived data*目录中，`Objects-normal`文件夹下找到这些文件。

查看`.o`文件的命令为：`查看命令 `xcrun size -x -l -m a.out``

```
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
	Section __text: 0x34 (addr 0x100000f50 offset 3920)
	Section __stubs: 0x6 (addr 0x100000f84 offset 3972)
	Section __stub_helper: 0x1a (addr 0x100000f8c offset 3980)
	Section __cstring: 0xe (addr 0x100000fa6 offset 4006)
	Section __unwind_info: 0x48 (addr 0x100000fb4 offset 4020)
	total 0xaa
Segment __DATA: 0x1000 (vmaddr 0x100001000 fileoff 4096)
	Section __nl_symbol_ptr: 0x10 (addr 0x100001000 offset 4096)
	Section __la_symbol_ptr: 0x8 (addr 0x100001010 offset 4112)
	total 0x18
Segment __LINKEDIT: 0x1000 (vmaddr 0x100002000 fileoff 8192)
total 0x100003000
```

###链接(Linke)

链接器解决了目标文件和库之间的链接。

`clang mian.m -o main`

生成可执行文件Executable

Where I will go on (应该是看看孙源推荐的基本资料开始探索吧)：

 Clang-LLVM 相关资料料
• <http://clang.llvm.org/docs/index.html>
• <http://blog.llvm.org/>
• <https://www.objc.io/issues/6-build-tools/compiler/>
• <http://llvm.org/docs/tutorial/index.html>
• <https://github.com/loarabia/Clang-tutorial>
• <http://lowlevelbits.org/getting-started-with-llvm/clang-on-os-x/>
• <https://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-i-introduction/>
• <http://szelei.me/code-generator/>
• 《Getting Started with LLVM Core Libraries》
• 《LLVM Cookbook》

