title: Introduction to JIT
date: 2014-11-12 11:12:41
categories: jvm
tags: JIT
---

JIT原理介绍。(转载)

<!--more-->

#### 什么是JIT

JIT，"Just In Time"的缩写，但是它究竟是什么意思呢？

> Whenever a program, while running, 
> **creates** and **runs** some new executable code
> which was not part of the program when it was stored on disk,
> it’s a JIT.

一句话：
> JIT就是在运行时动态创建并运行代码。

所以，JIT分为两个阶段：

1. 创建代码
2. 运行代码

创建代码的工作占了JIT的绝大部分，但是这部分比较好理解，编译器就是干这个的，如
`gcc`, `llvm`，都有生产代码的模块可供运行时调用。

运行代码，这部分不好理解，毕竟创建的代码是要经过OS同意才能运行的，并不是谁都
可以随随便便就能运行代码。OS运行任何应用都可以自由的创建数据，但是却禁止随意
运行代码。如何才能运行呢？使用`mmap`。

#### 一个简单的实例

比如我们要创建这样一段代码的机器码并运行，

```c
long add4(long num) {
    return num + 4;
}
```

下面是我们的JIT实现，

```c
// 分配内存空间
void* alloc_writable_memory(size_t size) {
    void* ptr = mmap(0, size,
                   PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (ptr == (void*)-1) {
        perror("mmap");
        return NULL;
    }
    return ptr;
}

// 设置执行权限
int make_memory_executable(void* m, size_t size) {
    if (mprotect(m, size, PROT_READ | PROT_EXEC) == -1) {
        perror("mprotect");
        return -1;
    }
    return 0;
}

// 主程序
void emit_to_rw_run_from_rx() {
    void* m = alloc_writable_memory(SIZE);
    emit_code_into_memory(m);
    make_memory_executable(m, SIZE);

    JittedFunc func = m;
    int result = func(2);
    printf("result = %d\n", result);
}
```

(over)

[文件来源及推荐阅读](http://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction)