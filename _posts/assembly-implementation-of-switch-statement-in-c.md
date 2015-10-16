title: switch语句的汇编实现
date: 2014-11-26 15:40:06
categories: 编译器
tags: 汇编
---

C语言中`switch`语句的汇编实现。

<!--more-->

一般来说，C语言中switch语句相当于一堆的if-else语句的累积，除非case很多的时候，
如果case之间的跨度不是很大，此时编译器一般会作优化，不再使用if-else集合，而是
使用一个跳转表，这样可以避免大量的compare计算。

#### 少量case时的switch语句

代码示例如下，

```c switch.c
#include <stdio.h>

void translate(int n) {
    char *p;
    switch(n) {
        case 1:
            p = "one";
            break;
        case 2:
            p = "two";
            break;
        case 3:
            p = "three";
            break;
    }
}
```

编译成汇编代码，

```bash
gcc -S switch.c
```

汇编代码如下，

```gas
# ignore some text ...
.LC0:
    .string "one"
.LC1:
    .string "two"
.LC2:
    .string "three"
.text
.global translate
    .type translate, @function
translate:
    # ignore some text
    # %eax keep the value of 'n'
    cmpl    $2, %eax
    je  .L4
    cmpl    $3, %eax
    je  .L5
    cmpl    $1, %eax
    jne .L1
.L3:
    movq $.LC0, -16(%rbp)   # -16(%rbp) is variable p
    jmp .L1
.L4:
    movq $.LC1, -16(%rbp)   # -16(%rbp) is variable p
    jmp .L1
.L5:
    movq $.LC2, -16(%rbp)   # -16(%rbp) is variable p
    nop
.L1:
    leave
    ret
# ignore some text...
```

这完全就是一堆的if-else集合。

#### 大量case时的switch语句

假如我们把上面的switch中的case数增加到8个，从`case 10:`一直到`case 17`，
则汇编代码变为，

```gas
# ignore some text
# %eax keep the value of 'n'
    subl    $10, %eax
    cmpl    $7, %eax
    ja  .L1 # 如果%eax > 7则跳转，
            # ja是无符号比较，所以即使n < 10，也就是说这里的%eax < 0，也会跳转
     # 根据跳转表进行跳转
     movq   .L11(,%eax,8), %rax
     jmp    *%rax
     .section   .rodata
     .align 8
     .align 4
.L11:
    .quad   .L3
    .quad   .L4
    .quad   .L5
    .quad   .L6
    .quad   .L7
    .quad   .L8
    .quad   .L9
    .quad   .L10
.L3: # 具体的case内容
    # xxx
.L4: # 具体的case内容
    # xxx
# ignore some text
    
```