title: xv6代码阅读：进程和进程调度
date: 2014-12-30 14:47:20
categories: [操作系统]
tags: [操作系统, xv6, process, scheduler]
---

xv6的进程结构以及进程调度。

<!-- more -->

### 基本数据结构
涉及到进程的有三大数据结构：`struct cpu`，`struct context`、`struct proc`。
`struct cpu`里面保存着当前CPU的相关信息，

```c proc.h
// Per-CPU state
struct cpu {
  uchar id;                    // Local APIC ID; index into cpus[] below
  struct context *scheduler;   // scheduler context，即scheduler运行环境信息
  struct taskstate ts;         // 用于interrupt时寻找进程的内核栈
  struct segdesc gdt[NSEGS];   // x86 global descriptor table
  volatile uint started;       // Has the CPU started?
  int ncli;                    // Depth of pushcli nesting.
  int intena;                  // Were interrupts enabled before pushcli?
  
  // Cpu-local storage variables
  struct cpu *cpu;
  struct proc *proc;           // The currently-running process.
};
```

`struct context`里面保存着运行环境信息，如果进程context，scheduler context，

```c
struct context {
  uint edi;
  uint esi;
  uint ebx;
  uint ebp;
  uint eip; // eip不会被显示设置，它在对swtch()函数的call和ret时被设置
};
```

`struct proc`里面保存着一个进程的信息，

```c
// Per-process state
struct proc {
  uint sz;                     // Size of process memory (bytes)
  pde_t* pgdir;                // Page table，页表，代表用户进程地址空间
  char *kstack;                // Bottom of kernel stack for this process，代表进程的内核栈
  enum procstate state;        // Process state
  volatile int pid;            // Process ID
  struct proc *parent;         // Parent process
  struct trapframe *tf;        // Trap frame for current syscall
  struct context *context;     // swtch() here to run process
  void *chan;                  // If non-zero, sleeping on chan
  int killed;                  // If non-zero, have been killed
  struct file *ofile[NOFILE];  // Open files
  struct inode *cwd;           // Current directory
  char name[16];               // Process name (debugging)
};
```

### 进程调度
进程调度分为两个部分：进程调出和进程调入。

- 调入
CPU会一直循环在`scheduler()`函数中，
符从ptable中寻找RUNNABLE进程，然后将其调入，即运行。
- 调出
调出也就是从用户态进程进入内核，如进程sleep, exit或者中断，调出在`sched()`中进行。

调入部分已经说过很多了，体现在`scheduler()`函数中，那么调出呢？

```c proc.c
void sched(void)
{
  int intena;

  if(!holding(&ptable.lock))
    panic("sched ptable.lock");
  if(cpu->ncli != 1)
    panic("sched locks");
  if(proc->state == RUNNING)
    panic("sched running");
  if(readeflags()&FL_IF)
    panic("sched interruptible");
  intena = cpu->intena;
  swtch(&proc->context, cpu->scheduler);
  cpu->intena = intena;
}
```

对比`scheduler()`函数，调入、调出其实就是在`swtch()`而已：

```c
// 调入
swtch(&cpu->scheduler, proc->context);
// 调出
swtch(&cpu->scheduler, proc->context);
```

调入就是：cpu->scheduler => proc->context
调出就是：proc->context => cpu->scheduler

swtch()的作用就是保存旧的context，然后切换到新的context并弹出寄存器值。

cpu指当前CPU，proc指当前进程。在调出和调入之间，需要kernel运行(不然谁来调啊)，
其实这就相当于有三者：old process context, kernel context, new process context。

当进程sleep、exit或者发生时钟中断时，当前进程就会被切换出去，当scheduler()找到它时，
再把它切换进来。

(over)