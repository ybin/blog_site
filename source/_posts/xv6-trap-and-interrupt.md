title: xv6 trap and interrupt
date: 2014-12-30 09:18:19
categories: [操作系统]
tags: [操作系统, xv6, trap, interrupt]
---

interrupt和trap在xv6上的实现，涉及到TSS，IDT等。

<!-- more -->

### interrupt, trap, exception是什么

### task state struct(TSS)是什么

### xv6是如何做的
每个进程，包括init进程，在创建之初就已经把该进程的kernel stack布局好了，

```c
/*
进程创建之初，其内核栈的样子：

                  /   +---------------+ <-- stack base(= p->kstack + KSTACKSIZE)
                  |   | ss            |                           
                  |   +---------------+                           
                  |   | esp           |                           
                  |   +---------------+                           
                  |   | eflags        |                           
                  |   +---------------+                           
                  |   | cs            |                           
                  |   +---------------+                           
                  |   | eip           | <-- 从此往上部分，在iret时自动弹出到相关寄存器中，只需把%esp指到这里即可
                  |   +---------------+    
                  |   | err           |  
                  |   +---------------+  
                  |   | trapno        |  
                  |   +---------------+                       
                  |   | ds            |                           
                  |   +---------------+                           
                  |   | es            |                           
                  |   +---------------+                           
                  |   | fs            |                           
 struct trapframe |   +---------------+                           
                  |   | gs            |                           
                  |   +---------------+   
                  |   | eax           |   
                  |   +---------------+   
                  |   | ecx           |   
                  |   +---------------+   
                  |   | edx           |   
                  |   +---------------+   
                  |   | ebx           |   
                  |   +---------------+                        
                  |   | oesp          |   
                  |   +---------------+   
                  |   | ebp           |   
                  |   +---------------+   
                  |   | esi           |   
                  |   +---------------+   
                  |   | edi           |   
                  \   +---------------+ <-- p->tf                 
                      | trapret       |                           
                  /   +---------------+ <-- forkret will return to
                  |   | eip(=forkret) | <-- return addr           
                  |   +---------------+                           
                  |   | ebp           |                           
                  |   +---------------+                           
   struct context |   | ebx           |                           
                  |   +---------------+                           
                  |   | esi           |                           
                  |   +---------------+                           
                  |   | edi           |                           
                  \   +-------+-------+ <-- p->context            
                      |       |       |                           
                      |       v       |                           
                      |     empty     |                           
                      +---------------+ <-- p->kstack             
 */
```

这些在init process的分析中都已经说过了，因为CPU执行`iret`时会把trapframe的eip
弹出到%eip寄存器中进而转到这里执行，所以我们只需设置trapframe的eip变量就可以
控制CPU的下一个执行指令地址，转到用户态之后进程的内核栈就是空的了。

接下来是trap的整个过程。我们以系统调用为例。

##### 从int $T_SYSCALL开始
进行系统调用的方式是这样的：

```as exec system call
  movl $SYS_exec, %eax
  int $T_SYSCALL
```

首先把trapno放到%eax里面，这个后续会用到。CPU在执行`int`指令时会自动跳转到相关的
中断处理程序里，也就是`vectorXXX`处，这里为每个中断设置了一个中断处理程序，xv6的
中断处理表是用vector.pl生成的，基本每个表项都是一样的，类似于这样，

```as
vector0:
  pushl $0  # errcode
  pushl $0  # T_SYSCALL
  jmp alltraps
```

`int`指令包含的内容是很多的，

1. 栈转换，stack转换到tr寄存器指定的tss里面的ss0:esp0，这是该进程对应的内核栈
2. 自动将相关寄存器的内容进栈：ss, esp(如果涉及到ring变化的话), eflag, cs, eip。
这样返回用户态之后才能恢复执行
3. 跳转到中断处理程序处

跳转完成之后，内核栈是这个样子的，

```c
/*
这部分是CPU执行int指令时自动压栈的，

 +---------------+ <-- stack base(= p->kstack + KSTACKSIZE)
 | ss            |                           
 +---------------+                           
 | esp           |                           
 +---------------+                           
 | eflags        |                           
 +---------------+                           
 | cs            |                           
 +---------------+                           
 | eip           |
 +---------------+ <-- %esp
 
 */
```

##### 中断处理程序
接着开始执行中断处理程序，在vectorXXX里面只是将errcode和trapno压栈，
errcode如果有的话都是0，trapno即系统调用的中断号T_SYSCALL，内核栈
变成这样了，

```c
/*
 +---------------+ <-- stack base(= p->kstack + KSTACKSIZE)
 | ss            |                           
 +---------------+                           
 | esp           |                           
 +---------------+                           
 | eflags        |                           
 +---------------+                           
 | cs            |                           
 +---------------+                           
 | eip           |
 +---------------+
 | err           |
 +---------------+
 | trapno        |
 +---------------+ <-- %esp
  */
```

然后jmp到alltraps继续执行，

```as
  # vectors.S sends all traps here.
.globl alltraps
alltraps:
  # Build trap frame.
  pushl %ds
  pushl %es
  pushl %fs
  pushl %gs
  pushal
  
  # Set up data and per-cpu segments.
  movw $(SEG_KDATA<<3), %ax
  movw %ax, %ds
  movw %ax, %es
  movw $(SEG_KCPU<<3), %ax
  movw %ax, %fs
  movw %ax, %gs

  # Call trap(tf), where tf=%esp
  pushl %esp
  call trap
  addl $4, %esp

  # Return falls through to trapret...
  # 这句话的意思是代码继续往下执行完trapret。
  # 'return'，名词；fall through to，动词，继续直到...
.globl trapret
trapret:
  popal
  popl %gs
  popl %fs
  popl %es
  popl %ds
  addl $0x8, %esp  # trapno and errcode
  iret
```

一进来就是一通压栈，压入段寄存器，压入通用寄存器`pushal`，到此为止，内核栈又
变样子了，

```c
/*
                  /   +---------------+ <-- stack base(= p->kstack + KSTACKSIZE)
                  |   | ss            |                           
                  |   +---------------+                           
                  |   | esp           |                           
                  |   +---------------+                           
                  |   | eflags        |                           
                  |   +---------------+                           
                  |   | cs            |                           
                  |   +---------------+                           
                  |   | eip           | <-- 从此往上部分，在iret时自动弹出到相关寄存器中，只需把%esp指到这里即可
                  |   +---------------+    
                  |   | err           |  
                  |   +---------------+  
                  |   | trapno        |  
                  |   +---------------+                       
                  |   | ds            |                           
                  |   +---------------+                           
                  |   | es            |                           
                  |   +---------------+                           
                  |   | fs            |                           
 struct trapframe |   +---------------+                           
                  |   | gs            |                           
                  |   +---------------+   
                  |   | eax           |   
                  |   +---------------+   
                  |   | ecx           |   
                  |   +---------------+   
                  |   | edx           |   
                  |   +---------------+   
                  |   | ebx           |   
                  |   +---------------+                        
                  |   | oesp          |   
                  |   +---------------+   
                  |   | ebp           |   
                  |   +---------------+   
                  |   | esi           |   
                  |   +---------------+   
                  |   | edi           |   
                  \   +---------------+ <-- %esp
 */
```

OMG，整个栈，此时不就是一个trapframe结构吗？是的！修改一下段寄存器之后开始调用
`trap()`函数，这是一个C函数，里面会处理所有的中断、trap，它的参数就是一个trapframe，

```as
  # 为trap()函数准备参数，struct trapframe*
  pushl %esp
  call trap
  # 干掉之前压栈的参数，完成之后%esp又指向trapframe结构了
  addl $4, %esp
```

到此为止，系统调用的实质工作已经完成了，接下来就要返回用户态了，也就是`trapret`，
`alltraps`会自动往下执行到`trapret`。很显然，它要把寄存器恢复回去，

```as
.globl trapret
trapret:
  popal     # 恢复通用寄存器
  popl %gs  # 恢复各种段寄存器
  popl %fs
  popl %es
  popl %ds
  addl $0x8, %esp  # trapno and errcode，干掉这两个
```

我们故意把`iret`分开来，在`iret`执行前，内核栈样子是这样的，

```c
/*
 +---------------+ <-- stack base(= p->kstack + KSTACKSIZE)
 | ss            |                           
 +---------------+                           
 | esp           |                           
 +---------------+                           
 | eflags        |                           
 +---------------+                           
 | cs            |                           
 +---------------+                           
 | eip           |
 +---------------+
 | err           |
 +---------------+
 | trapno        |
 +---------------+ <-- %esp
  */
```

咦，好眼熟啊，对这正是CPU执行`int`之后内核栈的样子，这部分是被CPU
自动压栈的，接下来是`iret`指令，又是一个很复杂的指令，它会把`int`
自动压栈的内容再自动恢复回去，这样程序就回到eip处了，即中断发生的
指令处，一切就跟没有发生一样。

可是系统调用，比如说read()，它的执行结果呢？

```c syscall.c
void syscall(void)
{
  int num;

  num = proc->tf->eax;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    proc->tf->eax = syscalls[num](); // 系统调用的执行结果
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            proc->pid, proc->name, num);
    proc->tf->eax = -1;
  }
}
```

原来如此，系统调用的执行结果保存在通用寄存器里了。

整个系统调用至此完结。

(over)