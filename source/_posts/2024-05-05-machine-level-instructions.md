---
title: 读书随想：机器指令
date: 2024-05-05 02:56:18
tags: [csapp, machine level]
mathjax: true
---

机器指令由**操作码（opcode）和操作数（operand）**组成。从定义的一般形式上了解机器指令并其组成部分，有助于我们站在更高的视角看待具体的各种机器指令。

<!-- more -->

## 机器指令

> 在{% post_link 'machine-level-code' 读书随想：机器级代码 %}一文中我们提到过，机器指令由**操作码（opcode）和操作数（operand）**组成，接下来我们也会通过这两方面来了解机器指令。

### 操作码

> 在最初的时候，我很困惑，CPU 处理各个类型的数据时，通过地址知道要从内存的某个位置开始，但如何知道要到哪个位置结束呢？或者说，CPU 如何知道处理的对象（即操作数）的大小。

x86-64 指令集包括完整的**针对字节、字、双字以及四字的指令**。大多数 GCC 生成的汇编代码指令都有一个字符的后缀，表明操作数的大小。例如，数据传送指令有四个变种：`movb`（传送字节）、`movw`（传送字）、`movl`（传送双字）和 `movq`（传送四字）。

由于是从 16 位体系结构扩展成 32 位的，Intel 用术语“字（word）”表示16位数据类型。因此，称 32 位数为“双字（double words）”，称 64 位数为“四字（quad words）”。

C 语言基本数据类型对应的 x86-64 表示如下：

| C声明|Intel 数据类型|汇编代码后缀|大小（字节）|
|--|--|--|--|
|`char`|字节|b|1|
|`short`|字|w|2|
|`int`|双字|l|4|
|`long`|四字|q|8|
|`char*`|四字|q|8|
|`float`|单精度|s|4|
|`double`|双精度|l|8|

> 理解后缀的含义有助于记忆，其中 `b` 表示 byte，`w` 表示 word， **`l` 表示 long word**（意思是长字，为啥不用 d 啊），`q` 表示 quad words。

> 与其说编译器根据数据类型的不同编译出不同后缀但效果相同的指令，不如说是根据数据大小的不同进行编译。指针类型和 `long` 类型都是四字，尽管类型不同，但却使用了相同的后缀。因此我们还会说，在机器指令中指针（或者说地址）和 long 类型的整数没有区别。

汇编代码也使用后缀 `l` 来表示 4 字节整数和 8 字节双精度浮点数。这不会产生歧义，因为浮点数使用的是一组完全不同的指令和寄存器。

> 然而根据搜索到的资料，以 `mov` 指令为例，`movss` 用于传送单精度浮点数，`movsd` 用于传送双精度浮点数。其中第一个 `s` 代表 scalar（标量），和向量相对；最后一个字母，`s` 代表 single，`d` 代表 double。
> 单精度浮点数的英文为 single-precision floating-point。


### 操作数

> 书中的原标题是“操作数指示符”，其实个人觉得指示符的说法很贴切。

大多数指令有一个或多个**操作数（operand）**：
1. 指示出执行一个操作中要使用的**源数据值**，源数据值可以以常数形式给出，或是从寄存器或内存中读出。
2. 指示出放置结果的**目的位置**，结果可以存放在寄存器或内存中。

因此，操作数分为三种类型：
1. **立即数（immediate）**，用来表示常数值。书写方式为 `$` 后面跟一个标准 C 表示法表示的整数，比如 `$-577` 或 `$0x1F`。汇编器会自动选择最紧凑的方式进行数值编码。
2. **寄存器（register）**，用来表示某个寄存器的内容。将寄存器集合看成一个数组 $R$，用寄存器标识符 $r_a$ 作为索引，用 $R[r_a]$ 来表示它的值。
3. **内存引用**，用来根据计算出来的**有效地址**访问某个内存位置。将内存看成一个很大的字节数组 $M$，用符号 $M_b[Addr]$ 表示存储在内存中从地址 $Addr$ 开始的 $b$ 个字节值的引用。为了简便，通常省去下标 $b$。

> 关于自动选择最紧凑的方式进行数值编码是指汇编器会根据数值的实际大小来决定使用多少个字节来存储该数值，避免不必要的空间浪费。

> 关于省去的下标 $b$，在实际情况中，$b$ 的大小由汇编指令的后缀指出。


#### 通用目的寄存器

一个 x86-64 的中央处理单元（CPU）包含一组 16 个存储 64 位值的**通用目的寄存器**（General-Purpose Registers，简称 GPRs）。

> 截图书中的表格可能已经足够形象易懂，但为了减少图片加载的开销，同时补充一些信息，本文直接使用了表格。个人最初的想法是通过了解命名的规则，可以帮助记忆和理解相应寄存器的用途。但是根据教授在课程中所说，目前这些寄存器的使用非常灵活，和命名没有联系。

|64<br/>register|32<br/>extended|16|8<br/>low|说明|命名原因|
|--|--|--|--|--|--|
|`%rax`|`%eax`|`%ax`|`%al`|返回值|accumulator register|
|`%rbx`|`%ebx`|`%bx`|`%bl`|被调用者保存|base register|
|`%rcx`|`%ecx`|`%cx`|`%cl`|第4个参数|count register|
|`%rdx`|`%edx`|`%dx`|`%dl`|第3个参数|data register|
|`%rsi`|`%esi`|`%si`|`%sil`|第2个参数|source index|
|`%rdi`|`%edi`|`%di`|`%dil`|第1个参数|destination index|
|`%rbp`|`%ebp`|`%bp`|`%bpl`|被调用者保存|base pointer|
|`%rsp`|`%esp`|`%sp`|`%spl`|栈指针|stack pointer|
|`%r8`|`%r8d`<br/>**double words**|`%r8w`<br/>**word**|`%r8b`<br/>**byte**|第5个参数|以下是在扩展到 x86-64 后新增|
|`%r9`|`%r9d`|`%r9w`|`%r9b`|第6个参数|
|`%r10`|`%r10d`|`%r10w`|`%r10b`|调用者保存|
|`%r11`|`%r11d`|`%r11w`|`%r11b`|调用者保存|
|`%r12`|`%r12d`|`%r12w`|`%r12b`|被调用者保存|
|`%r13`|`%r13d`|`%r13w`|`%r13b`|被调用者保存|
|`%r14`|`%r14d`|`%r14w`|`%r14b`|被调用者保存|
|`%r15`|`%r15d`|`%r15w`|`%r15b`|被调用者保存|

指令可以对这 16 个寄存器的低位字节中存放的不同大小的数据进行操作。

> 也是通过指令后缀指示。

有些指令可以复制和生成 1 字节、2 字节、4 字节和 8 字节值。当这些指令以寄存器作为目标时，对于生成小于 8 字节结果的指令，寄存器中剩下的字节会怎么样，对此有两条规则：

- 生成 1 字节和 2 字节结果的指令会保持剩下的字节不变。
- **生成 4 字节结果的指令会把高位 4 个字节置为 0**。

> 由于上述规则并非一个平时可以忽视的奇怪规则，难免让人好奇为何如此。保持剩余部分不变会有效率优势吗？置为 0 是为了什么呢？如果有不得不如此的理由，生成 1 字节和 2 字节时应该也不例外才对。
> 书中后续又有一处提到这是 x86-64 采用的惯例。教授在课程中表示不知道原因。网上未搜索到让人信服的答案。

如表格中的说明所示，在常见的程序里不同的寄存器扮演不同的角色，其中最特别的是栈指针 `%rsp`，用来指明运行时栈的结束位置。另外 15 个寄存器的用法更灵活。少量指令会使用某些特定的寄存器。**有一组标准的编程规范控制着如何使用寄存器来管理栈、传递函数参数、从函数的返回值，以及存储局部和临时数据**。

> 这里“从函数”一词是指被调用函数（callee）吗？

#### 内存引用

在内存引用的类型中，**寻址模式**的最常用形式为 $Imm(r_b, r_i, s)$。这样的引用有四个组成部分：

- 一个立即数偏移 $Imm$
- 一个基址寄存器 $r_b$
- 一个变址寄存器 $r_i$
- 一个比例因子 $s$

其中 $s$ 必须是 1、2、4 或者 8，基址和变址寄存器都必须是 64 位寄存器。

有效地址被计算为 $Imm + R[r_b] + R[r_i]·s$。引用数组元素时，会用到这种通用形式。其他形式都是这种通用形式的特殊情况，只是省略了某些部分，具体可参见下列表格。

|类型|格式|操作数值|名称|
|--|--|--|--|
|立即数|$\\$Imm$|$Imm$|立即数寻址|
|寄存器|$r_a$|$R[r_a]$|寄存器寻址|
|存储器|$Imm$|$M[Imm]$|绝对寻址|
|存储器|$(r_a)$|$M[R[r_a]]$|间接寻址|
|存储器|$Imm(r_b)$|$M[Imm + R[r_b]]$|（基址+偏移量）寻址|
|存储器|$(r_b, r_i)$|$M[R[r_b] + R[r_i]]$|变址寻址|
|存储器|$Imm(r_b, r_i)$|$M[Imm + R[r_b] + R[r_i]]$|变址寻址|
|存储器|$(, r_i, s)$|$M[R[r_i]·s]$|比例变址寻址|
|存储器|$Imm(, r_i, s)$|$M[Imm + R[r_i]·s]$|比例变址寻址|
|存储器|$(r_b, r_i, s)$|$M[R[r_b] + R[r_i]·s]$|比例变址寻址|
|存储器|$Imm(r_b, r_i, s)$|$M[Imm + R[r_b] + R[r_i]·s]$|比例变址寻址|

> 初见可能会感觉复杂，理清后其实很简单。对于这类严谨而繁琐的描述，初看厌烦，而后喜欢。


## 参考文章

- 《深入理解计算机系统》
