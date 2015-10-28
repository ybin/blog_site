title: xv6代码阅读：系统引导
date: 2014-12-24 14:27:03
categories: 操作系统
tags: [操作系统, xv6]
---

xv6的系统引导过程。这部分主要介绍从系统上电到进入C语言的世界(main())的过程，
内容包括引导扇区、内核img的加载、保护模式以及MMU的开启等。

<!-- more -->

### 启动扇区
系统上电之后，无论从硬盘启动还是从光驱启动，或者其他别的启动方式，BIOS都会把
启动扇区的512个字节复制到内存的0x7c00处，然后CPU从`%cs=0 %ip=0x7c00`处开始执行，
所以操作系统需要做的就是制作一个512字节大小的启动扇区，这里的代码做些必要的工作
之后就跳转到操作系统内核处开始执行，这样操作系统就开始运行起来了。

启动扇区的工作主要有：

1. enable A20
2. set `protection mode bit` in `cr0`
3. 进入32位模式(保护模式)并复制系统内核到内存
4. 跳转到内核代码继续执行

对于xv6来说，前两个阶段在`bootmasm.S`中完成，后两个在`bootmain.c`中完成。

##### 打开A20
什么是A20？详见维基百科[页面][A20]。
A20就是"Adress line 20"，以前的x86 CPU是16位的，但是Intel将其地址总线扩展
到20条，即A0~A19，这样内存寻址就能达到1MB(2^20)，由于其采用分段机制，即
段地址+段内偏移=实际内存地址，段内偏移用16位表示，而段寄存器也是16位的，
这样两者组合起来很容易就超过1MB，但是只有20条地址总线，于是超过1MB的部分
被自然而然的忽略掉了，即地址自动回旋，如`0x0010:0x0001`等于`0x0001`，
当时很多代码会依赖于此。

但是到了保护模式，地址总线达到32位，以上的两个地址不再相等，于是在real mode
下，第21位地址总线(A20)是被禁用的，这样两者再次等同。

如果要进入32位模式，必须要把A20打开，否则寻址能力达不到4GB。这就是原因所在。

```asm bootasm.S
# Physical address line A20 is tied to zero so that the first PCs 
# with 2 MB would run software that assumed 1 MB.  Undo that.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```

##### 启动保护模式
内存地址是段地址+段内偏移，在real mode下，段地址就放在段寄存器中，而到了保护模式下，
段寄存器中保存的不再是段地址，而是GDT(global descriptor table)中的偏移量，所以进入
保护模式时首先需要创建GDT。

GDT，全局描述符表，它类似于一个数组，每个项占用8个字节，每个段对应于一个表项，表项中
保存着段的基址，段的大小，访问权限等信息，每次访问内存时，CPU根据内存地址定位到具体
的段，然后根据这些信息检查内存访问是否合法，如往一个只读段写入数据就会导致CPU抗议进而
罢工，这就是保护模式名称的由来。

我们首先来创建GDT，它的第一个表项必须是空的，这是规定，作为一个临时的GDT，我们只为
code和data创建两个段即可，

```asm bootasm.S
# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULLASM                             # null seg
  SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
  SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg

gdtdesc:
  .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

然后就可以启动保护模式了，启动方式也很简单，只需把`cr0`寄存器中的`PE`位置位即可，

```asm bootasm.S
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0
```

##### 进入32位模式
保护模式已经启动，接下来通过一个`ljmp`跳转，`ljmp  $(SEG_KCODE<<3), $start32`，
`ljmp`的语法：`ljmp segment offset`，它会同时设置段寄存器和段内偏移(EIP寄存器)，
而在32位模式下，段寄存器保存的是GDT里的偏移量，所以这里段寄存器的值是`SEG_KCODE<<3`，
即数字8(SEG_KCODE等于1)，因为GDT的第一个表项必须为空，所以代码段占据第二个表项。

进入32位模式之后，设置各个段寄存器，数据段占用GDT第三个表项，ES, SS等同于数据段，
其他不用的段置零。

```asm
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
```

然后进入C语言的世界复制内核镜像，进入C的世界需要调用函数，必不可少的就是stack，
所以首先要设置stack，`movl $start, %esp`，即给esp寄存器设置，$start为bootasm.S
最开始的一个label，但是为何要将esp设置为$start呢，这是因为$start是代码的开始
位置，代码是向高内存地址处放置的，而stack是向低内存地址增长的，所以这样设置之后，
代码和栈向相反方向扩张，两者互不干涉。另外，当`call bootmain`的时候，首先要做的
就是`movl %esp %ebp`，即将esp的值设置给ebp，这样ebp保存的就是$start的值，也就是
函数栈中的第一个函数是`$start`，这符合逻辑。

##### 复制内核镜像
接下来进入C语言的世界，复制内核镜像，复制是通过IO port进行的，一个扇区一个扇区的
复制，这个我们不管，我们关注的是：内核镜像放在内存的什么位置。

内核镜像是ELF格式的，elf文件中保存着代码的物理内存地址，它规定了该把内核放置到
哪里，于是我们读取扇区数据，并按照elf格式解析数据，并将其放置到它要求的物理内存
处即可，这个物理内存是在链接脚本(kernel.ld)中设置的，即1MB处。

```
/* 内核入口点，保存在elf header的entry属性中 */
ENTRY(_start)

SECTIONS
{
    /* Link the kernel at this address: "." means the current address */
    /* Must be equal to KERNLINK */
    /* kernel的虚拟内存地址 */
	. = 0x80100000;

    /* kernel的物理内存地址，这个地址保存在elf->proghdr->paddr中，
	   从磁盘读取kernel代码时会用到(bootmain.c::bootmain()::pa变量) */
    /* 1MB，这跟kernel的memlayout正好对应起来，见memlayout.h */
	.text : AT(0x100000) {
		*(.text .stub .text.* .gnu.linkonce.t.*)
	}
	
	/* ........... */
}
```

也就是说，内核代码中用的都是虚拟地址，如main函数的地址就是`0x801xxxxx`，
`objdump -d kernel`看看就知道了，所有函数、变量都是虚拟地址，但是ENTRY保存的
却是物理地址，所以可以直接调用_start，而不用担心找不到它。

```c bootmain.c
  // 调用_start，进入内核的世界，它位于entry.S文件中
  entry = (void(*)(void))(elf->entry);
  entry();
```

到此为止，内存中的布局大致是这个样子的，

```c
/*
读取kernel数据到内存0x10000处，读取之后内存的样子如下:
0x10000(64KB)这个地方的内容只是暂时存放kernel img(elf文件)的hdr内容的，
根据elf header的内容进一步读取kernel img的内容，实际的内容将会存在在
1MB地址处，这个1MB地址是在kernel.ld中定义的(AT(0x100000))，这恰好跟
kernel memlayout吻合起来，见memlayout.h。

                   +-------------------+  4GB                 
                   |                   |                      
                   |                   |                      
                   |                   |                      
                   |                   |                      
                   |                   |                      
                   |                   |                      
                   |                   |                      
                   |                   |                      
                   +-------------------+                      
                   |                   |                      
 (main.c)main() -> |      kernel       |                      
                   |                   |                      
  0x100000(1MB) -> +-------------------+                      
                   |                   |                      
  0x10000(64KB) -> +elf hdr of kern img+    (tmp use. elf header content)                    
                   |                   |                      
   0x7c00 + 512 -> |      \x55\xAA     |                      
                   |                   |                      
       .gdtdesc -> +-------------------+                      
                   |                   |                      
           .gdt -> +-----+-------------+ <- gdtr(GDT Register)
                   |     |  seg null   |                      
                   | GDT |  seg code   |                      
                   |     |  seg data   |                      
                   +-----+-------------+                      
                   |                   |                      
                   |                   |                      
    bootmain()  -> |                   |                      
                   |        code       |                      
                   |                   |                      
      .start32  -> |                   |                      
                   |                   |                      
(0x7c00).start  -> +---------+---------+ <- esp               
                   |         |         |                      
                   |         v         |                      
                   |       stack       |                      
                   |                   |                      
                   |                   |                      
                   +-------------------+  0GB                 

 */
```

`\x55\xAA`是什么？它是启动扇区的标志，所有启动扇区最后的两个字节必须是这两个字节。

##### 启动扇区怎么制作呢？
如果生成启动扇区，如何生成内核镜像，这些要到Makefile文件中去寻找答案。

```Makefile
xv6.img: bootblock kernel fs.img
	dd if=/dev/zero of=xv6.img count=10000
	dd if=bootblock of=xv6.img conv=notrunc # bootblock部分放置到第一个扇区(该部分必须保证自己的size小于512bytes)
	dd if=kernel of=xv6.img seek=1 conv=notrunc # kernel代码放置到第二个以及以后的扇区
bootblock: bootasm.S bootmain.c
	$(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
	$(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o
	$(OBJDUMP) -S bootblock.o > bootblock.asm
	$(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
	./sign.pl bootblock # 检查bootblock的大小，并在最后两个字节处添加标志: 0x55 0xAA，这是启动扇区的标志。
kernel: $(OBJS) entry.o entryother initcode kernel.ld
	$(LD) $(LDFLAGS) -T kernel.ld -o kernel entry.o $(OBJS) -b binary initcode entryother
```

kernel就是内核镜像文件，而bootblock是启动扇区的内容。
系统引导部分至此结束，接下来从entry.S::_start开始分析内核代码。

(over)

[A20]: http://en.wikipedia.org/wiki/A20_line