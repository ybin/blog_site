title: 非本地跳转(NonLocal Jump)
date: 2014-11-12 16:26:11
categories: c/cpp
tags: [setjmp, longjmp]
---

正常的调用-返回流程是这样的，假如A调用B，B调用C，
则C返回到B，B返回到A，但是非本地跳转则打破了这种规则，
它可以不按照调用流程一步步返回，而是可以返回到任意地址，
例如完全可以从C中直接返回到A中，而越过B。

<!--more-->

#### 非本地跳转(setjmp)
非本地跳转由`setjmp.h`中的一组函数来实现。

```c
#include <setjmp.h>
int setjmp(jmp_buf env);
void longjmp(jmp_buf env, int retval);
```

`setjmp`函数最多可以返回2次，而`longjmp`函数则不返回。

`setjmp`函数相当于设置一个程序运行时点，第一次返回0表示设置运行时点成功，
第二次返回是由`longjmp`函数导致的，第二次的返回值则是`longjmp`函数的第二个参数值，
即`retval`。

`longjmp`函数永不返回，它只是迫使`setjmp`函数第二次返回，并设置其返回值。

`jmp_buf env`保存运行时点的状态。

`setjmp`函数和`longjmp`函数配合能达到类似于Java中的`try`、`catch`、`throw`的效果。

示例代码：

```c
#include <stdio.h>
#include <setjmp.h>

int E_FOO = 1;
int E_BAR = 2;
int E_UNKOWN = -1;

static jmp_buf env;

int main()
{
    int ret = 0;
    int rc;
    rc = setjmp(env);
    
    if(rc == 0) {                // 相当于 try block
        foo();
        bar();
        printf("run success\n");
    } else if(rc == E_FOO) {    // 相当于 catch(E_FOO)
        printf("error happened in foo()\n");
        ret = E_FOO;
    } else if(rc == E_BAR) {    // 相当于 catch(E_BAR)
        printf("error happened in foo()\n");
        ret = E_BAR;
    } else {                    // 相当于 catch(Exception)
        printf("error happened in foo()\n");
        ret = E_UNKOWN;
    }
}

void foo(void) {
    // longjmp(env, E_FOO);     // 相当于 throw(E_FOO)
}

void bar(void) {
    longjmp(env, E_BAR);        // 相当于 throw(E_BAR)
}
```

#### 带信号的非本地跳转(sigsetjmp)

对于POSIX系统来说，`setjmp`函数并不保存signal（BSD系统则会保存signal），
如果需要连同signal一并保存的话，需要使用另一组函数，

```c
#include <setjmp.h>
int sigsetjmp(sigjmp_buf env, int savesigs);
void siglongjmp(sigjump_buf env, int retval);
```

与`setjmp`不同的是`sigsetjmp`函数多了一个参数`savesigs`，如果它不为0，
则会将信号掩码保存到`env`中，否则不保存。

示例，

```c
#include <stdio.h>
#include <signal.h>
#include <setjmp.h>
#include <unistd.h>             // for sleep()

static sigjmp_buf env;

void handler(int sig) {
    siglongjmp(env, 1);         // return non-zero value
}

int main()
{
    signal(SIGINT, handler);    // 重置Ctrl-C信号
    
    int rc;
    rc = sigsetjmp(env, 1);     // 1: 保存信号掩码
    if(rc == 0) {
        printf("started\n");
    } else {
        printf("restarted\n");
    }
    
    int i = 0;
    while(i++ < 10)  {
        sleep(1); // sleep 1 second
        printf("processing...\n");
    }
    
    return 0;
}
```

程序输出类似于这样，

```bash
started
processing...
processing...
restarted        # 按下Ctrl-C
processing...
processing...
processing...
restarted        # 按下Ctrl-C
processing...
processing...
processing...
processing...
processing...
```

(over)