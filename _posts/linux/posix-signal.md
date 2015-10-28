title: 信号(signals)
date: 2014-11-17 10:14:43
categories: linux/unix
tags: 信号
---

[信号]()(signal)是\*nix以及POSIX兼容系统中的一种进程间通讯的方法。signal大致可以做如下划分：

- Traditional Unix signal
- POSIX standard signal
- POSIX real time signal

每一个信号对应一个整数值，每个信号都定义了默认行为，即进程收到信号之后默认执行的动作，
这些以及signal编程所需的其他内容都定义在`signal.h`中，可以通过`kill -l`命令查看都有
哪些signal，其中`SIGRTMIN`到`SIGRTMAX`之间的signal用于实时信号(real time signal)。有
多种方式可以产生信号，当然进程也有多种方式来处理信号：
- 使用默认行为
- 忽略信号(ignore)
- 捕获信号(catch)，即用户自定义signal handler
- 阻塞信号(block)

<!--more-->

#### 信号的产生

产生信号的方式有多种，

- 通过终端按键产生信号，如`Ctrl-C`产生`SIGINT`信号，`Ctrl-\`产生`SIGQUIT`信号
- 通过系统调用向进程发送信号
    - int kill(pid_t pid, int signo); // 向其他进程发送信号
    - int raise(int signo);           // 向当前进程发送信号(自己发给自己)
    - void abort(void);               // 向当前进程发送`SIGABRT`信号
- 通过软件产生信号，如通过`alart`系统调用产生`SIGALRM`信号

#### 传统的Unix信号

Unix信号使用方法比较简单，它只有一个接口，

```c
#include <signal.h>
void (*signal(int sig, void (*func)(int)))(int);

// 另一种比较容易理解的方式: 
// signal接受一个整数信号和一个func类型的函数，然后返回一个func类型的函数
typedef void (*func)(int);
func signal(int sig, func f);

// 忽略信号sig
signal(sig, SIG_IGN);
// 重置sig信号的默认行为
signal(sig, SIG_DFL);
```

Unix信号接口可以忽略、捕获信号，但是无法阻塞信号。

一个简单的实例：

```c Tradational Unix Signal
#include <signal.h>

static void signal_handler(int signo) {
    switch(signo) {
        case SIGINT:
            printf("You'v hit Ctrl+C.\n");
            signal(SIGINT, SIG_DFL);
        case SIGQUIT:
            printf("You'v hit Ctrl+\.\n");
            signal(SIGQUIT, SIG_DFL);
        default:
            printf("I don't know this signal.\n");
    }
}

int main() {
    // ...
    signal(SIGTTOU, SIG_IGN);
    signal(SIGTTIN, SIG_IGN);
    signal(SIGINT, signal_handler);
    signal(SIGQUIT, signal_handler);
    // ...
    return 0;
}
```

程序将会忽略`SIGTTOU`和`SIGTTIN`信号，捕获`SIGINT`和`SIGQUIT`信号，并设置
`signal_handler`函数为新的handler。在handler中，收到信号之后先打印出相关信息，
然后重置信号的默认行为。

整个过程大致是这样的(以用户按下`Ctrl+\`为例)：

1. 在`main`函数中注册`SIGQUIT`信号的处理函数为`signal_handler`
2. 程序执行过程中键盘中断到达，切换到内核态执行
3. 中断处理完毕后，在返回用户态之前检查发现有`SIGQUIT`信号(键盘驱动程序把Ctrl+\翻译为`SIGQUIT`信号)
4. 内核决定返回用户态执行`signal_handler`函数，它跟`main`函数使用不同的栈空间，这是两个独立的控制流程
5. `signal_handler`执行完毕自动执行特殊的系统调用`sigreturn`再次进入内核态
6. 内核再次检查信号，如果没有新的信号就恢复`main`函数的上下文继续执行

然而，Unix信号过于简单，很多场景下它会显得捉襟见肘，如，

- 在信号处理函数执行的过程中，新的信号到达该如何处理（特殊地，同样的信号再次到达）

针对这些问题，POSIX提出来他们的解决方案。

#### POSIX信号

POSIX信号兼容Unix信号。首先需要明确以下概念：

- 当信号出现时，我们用**产生**(generated)来表述
- 我们可以为信号定义**动作**
- 当信号对应的动作__执行__时，我们说信号被**送达**(delevered)了
- 从产生到送达，这期间信号处于**悬停**(pending)状态
- 一个进程的信号可以**阻塞**，如果该进程没有忽略该信号，那么该信号将处于pending状态
- 处于阻塞状态的信号可能多次产生，如果内核多次送达该信号，则称该信号被**入队**(queued)，如果只送达一次，则它没有被入队
- 每个进程都有一个bit array称为signal mask，表示哪些信号被阻塞：一个bit代表一个信号的阻塞状态，如果该bit为on则表示阻塞

##### POSIX标准信号

在内核中信号大致是这样的，

![Signal in Kernel](/res/img/signal_in_kernel.jpg)

为此，POSIX定义了一套接口用来处理信号，

###### signal set operations

signal set数据结构`sigset_t`用来表示一个信号集合，它用一个bit来表示一个信号
的on或者off，对应的有一套操作函数，

```c
#include <signal.h>
int sigemptyset(sigset_t *set);                     // clear set, set all bit off
int sigfillset(sigset_t *set);                      // fill set, set all bit on
int sigaddset(sigset_t *set, int signo);            // add signo into set
int sigdelset(sigset_t *set, int signo);            // delete signo from set
int sigismember(const sigset_t *set, int signo);    // does signo be member of set
```

###### sigprocmask函数

该函数可以读取或者改变进程的block mask，

```c
#include <signal.h>
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

参数`how`可以是
- `SIG_BLOCK`：效果相当于`mask |= set`
- `SIG_UNBLOCK`：效果相当于`mask &= ~set`
- `SIG_SETMASK`：效果相当于`mask = set`

如果`oldset`不为`NULL`则会保存之前的mask值，它可以用于恢复以前的状态。

###### sigpending函数

该函数用于读取当前pending状态的信号，

```c
#include <signal.h>
int sigpending(sigset_t *set);  // 处于pending状态的信号保存在set中
```

###### sigaction函数

该函数可以读取和修改与指定信号相关联的处理动作，

```c
#include <signal.h>
int sigaction(int signo, const struct sigaction *act, struct sigaction *oldact);

struct sigaction {
    void (*sa_handler)(int);    // signal handler or SIG_DFL or SIG_IGN
    sigset_t sa_mask;           // handler执行期间block mask设置为sa_mask，执行完之后自动恢复
    int sa_flags;               // TODO 待补充。。。
    void (*sa_sigaction)(int, siginfo_t *, void *);
}
```

###### sigsuspend函数

该函数重置进程block mask并挂起当前进程，注意，这两步是**原子操作**，这很重要。
函数返回后，block mask恢复为原来的值。

```c
#include <signal.h>
int sigsuspend(const sigset_t *mask);
```

写一个闹钟程序，每隔一段时间闹钟响一次，在闹钟响的间歇，进程挂起。

```c 闹钟程序 v1
#include <unistd.h> // for pause()
#include <signal.h>
#include <stdio.h>

void alarm_handler(int signo) { /* nothing to do */ }

void mysleep(unsigned int t) {
    struct sigaction newact, oldact;
    
    newact.sa_handler = alarm_handler;
    sigemptyset(&newact.sa_mask);           // unblock all signals
    newact.sa_flags = 0;
    sigaction(SIGALRM, &newact, &oldact);   // set user defined handler
    
    alarm(t);   // t seconds之后内核将会给进程发送SIGALARM信号
    pause();    // 设置好闹钟之后，进程挂起
    
    alarm(0);   // 取消闹钟
    sigaction(SIGALRM, &oldact, NULL);      // reset SIGALARM handler
}

int main() {
    while(1) {
        mysleep(1);
        printf("ONE second passed\n");
    }
    return 0;
}
```

仔细分析`mysleep`函数，该函数有个致命之处，如果在`alarm`函数和`pause`函数之间程序
暂停超过1s(可能被调度了)，那么就会导致调用`pause`之前，`SIGALAM`信号到达，然后执行
handler，之后才执行`pause`函数，如果在此之后没有其他信号送达，那么该进程就会永远
被挂起。为避免这种情况，我们必须要保证`SIGALRM`信号在`pause`之后到达，一个“解决”方法
是使用信号阻塞。其他不变，

```c 闹钟程序，使用signal block
// ...

// 1. 屏蔽SIGALRM信号
// 2. 调用alarm
alarm(t);
// 3. 接触信号屏蔽
// 4. 挂起
pause();

// ...
```

但是，信号在3, 4之间到达呢，仍然无法解决问题，如果能保证3, 4是原子操作，问题就解决了，
这正是`sigsuspend`函数的作用，

```c 闹钟程序，使用sigsuspend函数
void mysleep(unsigned int t) {
    struct sigaction newact, oldact;
    sigset_t newmask, oldmask, suspendmask;
    
    newact.sa_handler = alarm_handler;
    sigemptyset(&newact.sa_mask);           // unblock all signals
    newact.sa_flags = 0;
    sigaction(SIGALRM, &newact, &oldact);   // set user defined handler
    
    // block SIGALRM
    sigemptyset(&newmask);
    sigaddset(&newmask, SIGALARM);
    sigprocmask(SIG_BLOCK, &newmask, &oldmask);
    
    alarm(t);
    
    suspendmask = oldmask;
    sigdelset(&suspendmask, SIGALARM);      // unblock SIGALARM
    sigsuspend(&suspendmask);               // set block mast as suspendmask, and suspend
    
    alarm(0);
    sigaction(SIGALRM, &oldact, NULL);
    sigprocmask(SIG_SETMASK, &oldmask, NULL);
}
```

##### POSIX实时信号

待续。。。

---

#### 参考文档

1. [All about Linux signals](http://www.linuxprogrammingblog.com/all-about-linux-signals?page=show)
2. [Linux C编程一站式学习(pdf)](https://aboutc.googlecode.com/files/LinuxC.pdf)