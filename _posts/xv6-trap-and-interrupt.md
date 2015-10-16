title: xv6代码阅读：traps and interrupts
date: 2014-12-30 09:18:19
categories: [操作系统]
tags: [操作系统, xv6, trap, interrupt]
---

interrupt和trap在xv6上的实现，涉及到TSS，IDT等。

<!-- more -->

### interrupt, trap, exception是什么
在x86系统中，中断(interrupt)有如下分类，

1. CPU外部中断 - 外部设备产生的中断，称为**interrupt**
    1. 可屏蔽中断(interrupt，如IO设备)
    2. 不可屏蔽中断(NMI)，如磁盘错误，极少发生，一旦发生别无他法
2. CPU内部中断 - 由CPU本身产生的中断
	1. 软件主动产生的中断，称为**trap**
	2. CPU检测到的异常(如除数为0)，称为**exception**
    3. 潜在可恢复的错误，称为**fault**，如缺页故障
    4. 不可恢复的错误，称为**abort**

所有这些，interrupt, trap, exception，在CPU看来都是一样的，CPU称它们为**中断**，
所以interrupt有两个意思：泛指所有中断和特指外部中断，这需要根据上下文来区分。

### task state segment(TSS)是什么
TSS，即任务状态段，它是GDT/LDT中的一个段描述符，用来**提升**CPU的运行级别，要使用
这个段，CPU当前运行级别(CPL)必须小于等于该段的DPL，TSS段中保存的是任务状态，即一堆
的寄存器值，值得注意的是三组寄存器值：

1. ss0、esp0
2. ss1, esp1
3. ss2, esp2

这三组寄存器是用来寻找栈的，x86提供0~3共四个级别，而TSS用来**提升**级别，所以低级别
可以提升到高级别，提升之后需要把当前的任务状态保存起来，其实就是保存到ssX, espX指定
的栈中，比如从3提升到0，则CPU自动将任务状态保存到ss0:esp0指定的栈中，级别3这一组就
不需要了，因为它是最低的级别了。

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

原来如此，系统调用的执行结果保存在通用寄存器里了。那系统调用的参数呢？比如说打开文件时
的“文件路径”参数，这是怎么传递的呢？答案在syscall.c里面，

```c
int argint(int n, int *ip)
{
  // proc->tf->esp增加一个ret addr的偏移量即为参数位置
  return fetchint(proc->tf->esp + 4 + 4*n, ip);
}
```

process在创建之初就设置p->tf属性了，这个指针指向内核栈的某个位置，见`allocproc()`函数，
系统调用的参数是通过读取用户栈传递的。

整个系统调用至此完结。

### 疑问及解答
从scheduler()看来，每个CPU都会找RUNNABLE的进程执行，而一旦开始执行就会将进程状态设置为
RUNNING，于是其他CPU不可能再执行这个进程了，这样看来，一个进程**同时**只能被一个CPU执行！
当然，也有可能进程A被CPU0执行一段时间，然后不管是被中断还是时间片用完，还是自己主动yield，
它都会再次进入RUNNABLE状态，然后其他CPU可能发现并执行它，但是**同一时刻**，它只能在一个
CPU上执行。但是问题来了，一个进程在CPU0上被中断，然后还能在CPU1上被恢复吗？

答：
这涉及到抢占式内核和非抢占式内核之分，如果进程在内核态执行时运行进程被更高优先级进程挂起，
那么kernel就是抢占式的，否则一个进程在进入kernel执行时不允许被中断，那么kernel就是非抢占式的。

xv6是抢占式的内核，也就是说，当进程通过系统调用进入内核之后允许中断，当进程通过系统调用进入内核
正在运行，此时中断再次到达后，比如时钟中断，它会再次进入到alltraps，在这里push各种寄存器，
然后再次执行到trap，并主动yield()，这个过程其实类似于调用`int`指令，只是这是发生在内核中的
中断，CPU自动保存寄存器时不会保存ss, esp(trapframe最后两个变量)，这之后当前正在运行的进程
内核栈中就保存了一个trapframe，一个context，而trapret会return到新的进程继续执行，因为这期间
栈被切换了。等待这个进程被再次调度后会从它的context继续执行，而此次执行就不一定是以前的那个
CPU了。

(over)