title: xv6代码阅读：init进程的启动
date: 2014-12-26 14:46:01
categories: 操作系统
tags: [操作系统, xv6, init]
---

xv6第一个进程的启动过程。涉及从系统引导进kernel到kernel的初始化，然后启动第一个进程
的整个过程。

<!-- more -->

上一节系统引导分析了整个引导过程，接下来从kernel入口`_start`(entry.S)往下分析，主要
涉及到启动分页，内核初始化，启动多处理器，启动init进程。

### 启动分页
分页机制或者说MMU是什么不作介绍，这里说下分页机制的一些必要操作。
启动分页之前必须创建页表并设置给cr3寄存器，然后给cr0寄存器的PG位置为1。除此之外，
x86还允许创建不同粒度的内存页，这涉及到cr4寄存器。

在x86中，一个也目录中可以同时存在两种粒度的内存页，4K或者4M，

![X86 4KB page](/res/img/X86_Paging_4K.png)

![X86 4MB page](/res/img/X86_Paging_4M.png)

page dir是必须的，这是一个长度**最大**为1024的整数数组，如果页目录表项的PS位置为1
且cr4寄存器的PSE位置为1，那么CPU自动使用4M大小的内存页，即该页目录表项中保存的就是
内存页的起始地址，这相当于进行二级分页而不是更常见的三级分页。如果这两个要求不能同时
满足就进行三级分页。

注意x86运行两种分页同时存在，比如cr4的PSE位设为1，而有些page dir表项设置PS位而有些
则不设置，这样就同时存在两种分页机制。为何要使用4MB页呢，考虑这种场景：kernel img
大小大约为1M，如果使用4M页映射kernel img，则TLB只需缓存一个页目录项即可，而如果是
4K页则需要256个页目录项，这么多的表项是无法都缓存到TLB中的，这会使得地址翻译变慢很多。
所以kernel img部分一般用一个4M页进行映射，而其他则使用4K页。

对于xv6来说，使用4M页只是临时，不用创建复杂的页表，如此而已，
内核启动之后很快就会重新创建页表。

```asm entry.S
.globl _start
_start = V2P_WO(entry) # kernel的入口仍然是低地址值，因为此时分页还没有开启

# Entering xv6 on boot processor, with paging off.
.globl entry
entry:
  # 设置cr4，使用4M页，这样创建的页表比较简单
  movl    %cr4, %eax
  orl     $(CR4_PSE), %eax
  movl    %eax, %cr4
  # 设置cr3
  movl    $(V2P_WO(entrypgdir)), %eax
  movl    %eax, %cr3
  # 启动分页
  movl    %cr0, %eax
  orl     $(CR0_PG|CR0_WP), %eax
  movl    %eax, %cr0

  # 创建CPU栈，这个是该CPU独有的，启动其他CPU时，每个都自己的stack
  movl $(stack + KSTACKSIZE), %esp
  
  # 进入高地址空间(2GB以上)
  mov $main, %eax
  jmp *%eax

# common symbol，开辟stack区域，大小为KSTACKSIZE
.comm stack, KSTACKSIZE
```

我们来看看页目录`entrypgdir`是什么样子的，

```c main.c
pde_t entrypgdir[NPDENTRIES] = {
  // Map VA's [0, 4MB) to PA's [0, 4MB)
  [0] = (0) | PTE_P | PTE_W | PTE_PS,
  // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
  [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```

疑问：为何要将0~4M进行1:1映射呢？在开启分页之前我们都是小心翼翼的使用“低地址”，
而打开分页之后我们将会跳转到“高地址”，低地址还有必要映射吗？有必要。在启动多处理器
的时候，还需要从低地址启动，因为这些CPU（non-boot CPU，也叫做AP）要需要从real mode
启动，见entryOther.S

### 内核初始化
到现在为止，只有boot CPU在运转，其他CPU需要boot CPU去启动才行，在启动之前，boot
CPU先要作必要的初始化工作。

```c
int
main(void)
{
  // 初始化kernel img结束位置直到4M之间的虚拟内存，kernel code
  // 从1MB处开始，大约有1MB的大小，所以剩余2MB可以用来kalloc
  kinit1(end, P2V(4*1024*1024)); // phys page allocator
  kvmalloc();      // kernel page table
  mpinit();        // collect info about this machine
  lapicinit();
  seginit();       // set up segments
  cprintf("\ncpu%d: starting xv6\n\n", cpu->id);
  picinit();       // interrupt controller
  ioapicinit();    // another interrupt controller
  consoleinit();   // I/O devices & their interrupts
  uartinit();      // serial port
  pinit();         // process table
  tvinit();        // trap vectors
  binit();         // buffer cache
  fileinit();      // file table
  iinit();         // inode cache
  ideinit();       // disk
  if(!ismp)
    timerinit();   // uniprocessor timer
  startothers();   // start other processors
  // 初始化4M直到PHYSTOP部分的内存
  kinit2(P2V(4*1024*1024), P2V(PHYSTOP)); // must come after startothers()
  userinit();      // first user process
  // Finish setting up this processor in mpmain.
  mpmain();
}
```

初始化部分其实是很复杂的，牵涉到各个部分，现在我们只是简单的提及，后续做详细介绍。
`kinit1()`，初始化`end~4MB`这一段内存，这样`kalloc()`就能分配出内存来了。
`kvmalloc()`重建内核页表并设置cr3切换到该页表，该页表会把IO空间，内核镜像等都建立
映射，这些空间全都是用户控件可见的，位于2GB以上，其定义如下，

```c vm.c
static struct kmap {
  void *virt;
  uint phys_start;
  uint phys_end;
  int perm;
} kmap[] = {
 { (void*)KERNBASE, 0,             EXTMEM,    PTE_W}, // I/O space
 { (void*)KERNLINK, V2P(KERNLINK), V2P(data), 0},     // kern text+rodata
 { (void*)data,     V2P(data),     PHYSTOP,   PTE_W}, // kern data+memory
 { (void*)DEVSPACE, DEVSPACE,      0,         PTE_W}, // more devices
};
```

`seginit()`初始化段寄存器，注意，logical address指的是"段内地址"，
virtual address又叫做线性地址，这里的设置把所有段(除gs段)基址均设为0，
这样实际就取消分段机制了。

```c vm.c
/*
 cpus[] --> +------------------------+                +---------+ <-- %gs of CPU0
            |                        |                | %gs:0~3 |                
            |         ......         |                +---------+                
            |                        |                | %gs:3~7 |                
      +---> +------------------------+                +---------+                
      |     |      uchar id          |                                           
      |     |    struct context*     |                +---------+ <-- %gs of CPU1
      |     |  struct segdesc gdt[]  |      +-------+ | %gs:0~3 |                
      |     |        ......          |      |         +---------+                
      |     +------------------------+      | +-----+ | %gs:3~7 |                
      +---- |    struct cpu *cpu     | <----+ |       +---------+                
            +------------------------+        |                                  
            |   struct proc *proc    | <------+                                  
            +------------------------+                +---------+ <-- %gs of CPU2
            |                        |                | %gs:0~3 |                
            |         ......         |                +---------+                
            |                        |                | %gs:3~7 |                
            +------------------------+                +---------+                
 */
void
seginit(void)
{
  struct cpu *c;

  // Map "logical" addresses to virtual addresses using identity map.
  // Cannot share a CODE descriptor for both kernel and user
  // because it would have to have DPL_USR, but the CPU forbids
  // an interrupt from CPL=0 to DPL=3.
  c = &cpus[cpunum()];
  c->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
  c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
  c->gdt[SEG_UCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
  c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);

  // Map cpu, and curproc
  // 基址: struct cpu的cpu属性地址，参考struct cpu的定义，
  // cpu、proc两个属性恰好一前一后挨着。
  c->gdt[SEG_KCPU] = SEG(STA_W, &c->cpu, 8, 0);

  lgdt(c->gdt, sizeof(c->gdt));
  loadgs(SEG_KCPU << 3);
  
  // Initialize cpu-local storage.
  cpu = c;
  proc = 0;
}

// proc.h
extern struct cpu *cpu asm("%gs:0");       // &cpus[cpunum()]
extern struct proc *proc asm("%gs:4");     // cpus[cpunum()].proc
```

### 启动多处理器
多处理器的启动是通过`IPI`，即Inter-Processor Instructions进行的，这是CPU间的通讯方式。
`startothers()`来启动其他non-boot CPU，做法是：

1. 复制启动代码到`0x7000`处，这部分代码相当于boot CPU的启动扇区代码
2. 为每个AP分配stack（是的，每个CPU都一个自己的stack）
3. 告诉每个AP，kernel入口在哪里(mpenter函数)
4. 告诉每个AP，页目录在哪里(entrypgdir)

然后控制local apic进行CPU间通讯，依次启动其他CPU。启动之后他们会执行mpenter()，进而进入
scheduler()开始执行程序。

### 启动init进程
boot CPU启动其他CPU之后，自己继续执行`kinit2()`初始化剩余的内存空间，然后开始启动init进程。
在`userinit()`中，

```c
void userinit(void)
{
  struct proc *p;
  extern char _binary_initcode_start[], _binary_initcode_size[];
  // 分配进程数据结构并初始化
  p = allocproc();
  initproc = p;
  // 创建页目录，也就是说创建进程地址空间，setupkvm()会将kernel部分映射进来
  if((p->pgdir = setupkvm()) == 0)
    panic("userinit: out of memory?");
  // 分配用户空间物理内存，并映射到虚拟内存的0处
  inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
  p->sz = PGSIZE;
  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0;  // beginning of initcode.S

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  p->state = RUNNABLE;
}
```
##### 创建进程数据结构
首先创建进程数据结构，然后初始化，并设其状态为RUNNABLE，等待进入scheduler()
执行，这里需要特殊说明的是`allocproc()`函数，它在全局变量`ptable`中寻找UNUSED的进程结构，
如果找到就做必要的初始化，然后将其返回，否则返回0，即空指针。其初始化过程包括修改进程状态，
设置进程pid，构建进程的kenel stack（每个进程都有一个对应的内核栈）。内核栈的构建是这样的，

```c proc.c
  // p为找到的进程结构
  if((p->kstack = kalloc()) == 0){
    p->state = UNUSED;
    return 0;
  }

  // stack逆向增长
  sp = p->kstack + KSTACKSIZE;
  
  // Leave room for trap frame.
  sp -= sizeof *p->tf;
  p->tf = (struct trapframe*)sp;
  
  // Set up new context to start executing at forkret,
  // which returns to trapret.
  sp -= 4;
  *(uint*)sp = (uint)trapret;

  // set up context
  sp -= sizeof *p->context;
  p->context = (struct context*)sp;
  memset(p->context, 0, sizeof *p->context);
  p->context->eip = (uint)forkret;

  return p;
```

每个进程的创建都是这样的，不论是init进程还是fork创建的普通进程。进程的内核栈到底什么样子呢，
图示如下，

```c
/*
进程的kernel stack初始化状态，

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

当然，这样的初始化结果是经过精心设计的，比如`trapret()`、`forkret()`函数地址的设置。

注意trapframe里的eip正是进入用户态之后执行的程序的入口，init进程跟普通进程
的差别就在这里了，init进程设置的eip=0，在此之前0这里已经放置了initcode.S的
内容，而普通进程这里设置的是elf->entry，即程序的main()函数。

##### 构建进程地址空间
接下来为进程创建页目录，也就是构建进程地址空间，这里会把kernel也映射到
进程空间，

```c vm.c
void
inituvm(pde_t *pgdir, char *init, uint sz)
{
  char *mem;
  if(sz >= PGSIZE)
    panic("inituvm: more than a page");
  // 分配用户态物理内存
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  // 映射到用户虚拟地址 0 处
  mappages(pgdir, 0, PGSIZE, v2p(mem), PTE_W|PTE_U);
  // 准备init进程的执行代码
  memmove(mem, init, sz);
}
```

##### 进入调度器开始执行
boot CPU继续执行进入`scheduler()`函数，它线性遍历ptable寻找
RUNNABLE的进程，找到之后就执行，无穷循环，

```c 开始执行进程p
  proc = p; // proc变量记录当前CPU正在执行的进程
  switchuvm(p);
  p->state = RUNNING;
  // swtch返回到forkret函数中，见p->kstack的context设置， 以及swtch.S
  swtch(&cpu->scheduler, proc->context);
  switchkvm();

  // Process is done running for now.
  // It should have changed its p->state before coming back.
  proc = 0;
```

进程数据结构创建之后就做好准备被调度了，在scheduler()中，首先要进行user vm
的设置，即`switchuvm(p)`，它会设置TSS段并且将其中的ss0设置为SEG_KDATA，把
esp0设置成p->kstack+KSTACKSIZE，也就是该进程内核栈的栈底(stack base，上图)，
然后`ltr(SEG_TSS<<3)`，这会让CPU在执行`iret`时使用该TSS的内容。设置ss0, esp0
的意思就是说，该进程以后被中断或者trap时，无论什么情况，只要进入kernel，那么
就使用这个stack，也就是说告诉进程它的返程路线是什么。

然后切换到进程页目录，注意，因为kernel空间也映射进用户也目录了，
所以kernel在切换页目录之后依然正常执行，标记为进程状态之后开始启动进程，看`swtch()`，

```asm swtch.S
.globl swtch
swtch:
  movl 4(%esp), %eax  # cpu context
  movl 8(%esp), %edx  # proc context

  # Save old callee-save registers
  # 旧的context进栈，进的是是CPU的栈，不是进程内核栈
  pushl %ebp
  pushl %ebx
  pushl %esi
  pushl %edi

  # Switch stacks
  movl %esp, (%eax) # %esp此时执行上面四个寄存器信息，现在给cpu的context变量赋值
  movl %edx, %esp   # %esp指向新的proc的context，上图中的p->context

  # Load new callee-save registers
  popl %edi
  popl %esi
  popl %ebx
  popl %ebp
  # popl之后，剩下的是eip，见上图，ret会返回"ret addr"处继续执行，也就是返回到forkret()继续执行
  ret
```

`forkret()`启动了一个log之后就返回了，接下来在栈中的是`trapret()`的地址，于是返回到`trapret()`
继续执行，

```asm trapasm.S
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

继续看上面的图，`popal`弹出一堆寄存器值，`popl`又弹出一堆，然后`addl $0x8, %esp`越过
trapno和errcode，接下来的就成了`eip`了(见`struct trapframe`)，然后`iret`(interrupt return)，
返回到`eip`指定的地址继续执行，回到`userinit()`函数，

```c
  memset(p->tf, 0, sizeof(*p->tf));
  // 进入用户态之后使用这些值
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE; // 用户栈从page size处逆向增长
  p->tf->eip = 0;  // beginning of initcode.S，用户态代码
```

我们回到用户空间的0处执行，而此处已经被`inituvm()`放置上initcode.S的内容了，
也就是说接下来执行initcode.S的内容。这个文件很简单，通过系统调用exec执行`/init`程序而已。

```c
/*
 init进程的内存布局：
 
 +--------------------+ 4GB                        
 |                    |                            
 |                    |                            
 |                    |                            
 +--------------------+ KERNBASE+PHYSTOP(2GB+224MB)
 |                    |                            
 |   direct mapped    |                            
 |   kernel memory    |                            
 |                    |                            
 +--------------------+                            
 |    Kernel Data     |                            
 +--------------------+ data                       
 |    Kernel Code     |                            
 +--------------------+ KERNLINK(2GB+1MB)          
 |   I/O Space(1MB)   |                            
 +--------------------+ KERNBASE(2GB)              
 |                    |                            
 |                    |                            
 |                    |                            
 |                    |                            
 |                    |                            
 |                    |                            
 |                    |                            
 |                    |                            
 +---------+----------+ PGSIZE <-- %esp                            
 |         v          |                            
 |       stack        |                            
 |                    |                            
 |                    |                            
 |     initcode.S     |                            
 +--------------------+ 0  <-- %eip               
*/
```

(over)