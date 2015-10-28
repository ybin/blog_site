title: x86 寄存器
date: 2014-12-20 11:46:17
categories: 编程
tags: [x86, 寄存器]
---

X86家族CPU寄存器介绍。

<!-- more -->

x86家族CPU寄存器比较少，整理起来也比较方便，但是他们仍然可以分门别类，
现整理如下。

### 通用寄存器(general registers)

按照位数的不同，通用寄存器进一步分为8位、16位、32位通用寄存器，

32 bits: EAX, EBX, ECX, EDX
16 bits: AX, BX, CX, DX
8  bits: AH AL, BH BL, CH CL, DH DL

其中，"E"表示extended，"H"表示high byte，"L"表示low byte。
A、B、C、D四组通用寄存器其实也是有自己的专长的，而不仅仅是按照字母顺序命名而已，

EAX, AX, AH, AL:
`Accumulator register`，它们用于I/O port access, arithmetic, interrupt calls etc.
EBX, BX, BH, BL:
`Base register`，它们被当做base pointer for memory access
ECX, CX, CH, CL:
`Counter register`，它们被当做loop counter and for shifts
EDX, DX, DH, DL:
`Data register`，它们用于I/O port access, arithmetic, some interrupt etc.

### 段寄存器(segment registers)

CS: code segment
DS: data segment
ES, FS, GS: extra segment. 用于far pointer addressing like video memory and such
SS: stack segment

### 索引和指针寄存器(index and pointers)

ES:EDI, EDI, DI:
Destination index register. Used for string, memory array copying and setting
and for far pointer addressing with ES

DS:ESI, EDI, SI:
Source index register. Used for string and memory array copying

SS:EBP, EBP, BP:
Stack Base pointer register. Holds the base address of the stack
                
SS:ESP, ESP, SP:
Stack pointer register. Holds the top address of the stack

CS:EIP, EIP, IP:
Index Pointer. Holds the offset of the next instruction. It can `only` be read

### 标志位寄存器(flag register)

标志位寄存器只有一个: EFLAGS，各个bit的含义如下，

```c
/*
Bit   Label    Desciption
---------------------------
0      CF      Carry flag
2      PF      Parity flag
4      AF      Auxiliary carry flag
6      ZF      Zero flag
7      SF      Sign flag
8      TF      Trap flag
9      IF      Interrupt enable flag
10     DF      Direction flag
11     OF      Overflow flag
12-13  IOPL    I/O Priviledge level
14     NT      Nested task flag
16     RF      Resume flag
17     VM      Virtual 8086 mode flag
18     AC      Alignment check flag (486+)
19     VIF     Virutal interrupt flag
20     VIP     Virtual interrupt pending flag
21     ID      ID flag

Those that are not listed are reserved by Intel.
 */
```

### 控制寄存器(controll registers)

CR0: 开启/关闭 保护模式
CR1:
CR2:
CR3:
CR4:

### 调试寄存器(debug registers)

DR0 ~ DR7

### 测试寄存器(test registers)

TR3 ~ TR7

### 其他寄存器

GDTR: Global Descriptor Table Register
LDTR: Local Descriptor Table Register
IDTR: Interrupt Descriptor Table Register
TR:

这篇文章大部分内容来自[这里][x86-registers]，致谢。


[x86-registers]: http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html