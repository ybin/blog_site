title: JamVM如何实现Stop-The-World
date: 2014-11-14 16:30:17
categories: jvm
tags: [JamVM, JVM]
---

在垃圾回收时，JamVM使用signal实现stop-the-world。

<!--more-->

#### 实现机制

- Thread会捕获`SIGUSR1`信号，并且有一个标志位`suspend`(boolean)标识线程的状态
- stop-the-world时，gc线程把其他所有线程的`suspend`标志位置为`TRUE`，并且发送`SIGUSR1`信号
- 在`SIGUSR1`的handler中，线程检查`suspend`标识，如果为`TRUE`就自动挂起，并继续等待`SIGUSR1`信号
- resume-the-world时，gc线程把其他所有线程的`suspend`标志位置为`FALSE`，并且发送`SIGUSR1`信号
- 此时在`SIGUSR1`的handler中，线程检查`suspend`标识为`FALSW`，于是恢复执行

#### 代码解读

代码位置： `thread.c`

##### 初始化signal

在VM初始化时，初始化signal，

```c
static int initialseSignals() {
    struct sigaction act;
    // 屏蔽SIGQUIT, SIGINT, SIGPIPE信号，子线程自动继承该mask
    initialiseSignalMask();
    
    // 捕获SIGUSR1信号，设置新handler为suspendHandler，该设置为process-wide，
    // 故所有线程均会捕获该信号
    act.sa_handler = suspendHandler;
    sigempty(&act.sa_mask);
    act.sa_flags = SA_RESTART;
    sigaction(SIGUSR1, &act, NULL);
    
    // ...
}
```

然后我们看看`suspendHandler`函数，

```c
static void suspendHandler(int sig) {
    Thread *thread = threadSelf();
    suspendLoop(thread);
}

static void suspendLoop(Thread *thread) {
    char old_state = thread->suspend_state;
    sigjmp_buf env;
    sigset_t mask;

    sigsetjmp(env, FALSE);

    thread->stack_top = &env;
    thread->suspend_state = SUSP_SUSPENDED;
    MBARRIER();

    sigfillset(&mask);
    sigdelset(&mask, SIGUSR1);
    sigdelset(&mask, SIGTERM);

    /*
       屏蔽除SIGUSR1, SIGTERM外的所有信号。
       如果suspend为TRUE，挂起
       如果suspend为FALSE，执行为该函数后自动跳转到线程正常代码，即恢复
     */
    while(thread->suspend && old_state == SUSP_NONE)
        sigsuspend(&mask);

    thread->suspend_state = old_state;
    MBARRIER();
}
```

##### Stop the world

要想stop the world，只需做两件事情：

1. 设置线程的`suspend`标志位为`TRUE`
2. 向该线程发送`SIGUSR1`信号

代码如下，

```c
void suspendAllThreads(Thread *self) {
    Thread *thread;

    pthread_mutex_lock(&lock);

    // 遍历所有其他线程，做两件事情：
    for(thread = &main_thread; thread != NULL; thread = thread->next) {
        if(thread == self)
            continue;

        // 1. 设置suspend标志位
        thread->suspend = TRUE;
        MBARRIER();

        // 发送SIGUSR1信号，这会是的thread线程在捕获信号后挂起
        if(thread->suspend_state == SUSP_NONE) {
            TRACE("Sending suspend signal to thread %p id: %d\n",
                  thread, thread->id);
            if(pthread_kill(thread->tid, SIGUSR1) == ESRCH) {
                /* ESRCH indicates that the thread has died.  This can only
                   occur when an external thread has been attached to the VM
                   via JNI and it has exited without detaching.  Although it
                   is a user error, it will deadlock the suspension code as it
                   will hang waiting for the thread to suspend.  Set the state
                   to BLOCKING, to ignore the thread. Note, no attempt is made
                   to clean-up the error; the thread will still appear to be
                   "live" (as with Hotspot).  We simply stop the thread from
                   hanging the VM. */
                TRACE("Setting thread %p id: %d state to BLOCKING "
                      "as it has died\n", thread, thread->id);
                thread->suspend_state = SUSP_BLOCKING;
            }
        }
    }

    // 等待线程挂起，(然后才能gc)
    for(thread = &main_thread; thread != NULL; thread = thread->next) {
        if(thread == self)
            continue;

        while(thread->suspend_state != SUSP_BLOCKING &&
              thread->suspend_state != SUSP_SUSPENDED) {
            TRACE("Waiting for thread %p id: %d to suspend\n",
                  thread, thread->id);
            sched_yield();
        }
    }

    all_threads_suspended = TRUE;

    TRACE("All threads suspended...\n");
    pthread_mutex_unlock(&lock);
}
```

##### Resume the world

要想resume the world，同样做两件事情：

1. 设置线程的`suspend`标志位为`FALSE`
2. 向该线程发送`SIGUSR1`信号

代码如下，

```c
void resumeAllThreads(Thread *self) {
    Thread *thread;

    TRACE("Thread %p id: %d is resuming all threads\n", self, self->id);
    pthread_mutex_lock(&lock);

    // 要想resume the world，只需做两件事：
    for(thread = &main_thread; thread != NULL; thread = thread->next) {
        if(thread == self)
            continue;

        // 1. 设置线程标志位
        thread->suspend = FALSE;
        MBARRIER();

        // 向该线程发送信号
        if(thread->suspend_state == SUSP_SUSPENDED) {
            TRACE("Sending resume signal to thread %p id: %d\n",
                  thread, thread->id);
            pthread_kill(thread->tid, SIGUSR1);
        }
    }

    // 等待线程恢复
    for(thread = &main_thread; thread != NULL; thread = thread->next) {
        while(thread->suspend_state == SUSP_SUSPENDED) {
            TRACE("Waiting for thread %p id: %d to resume\n",
                  thread, thread->id);
            sched_yield();
        }
    }

    all_threads_suspended = FALSE;
    if(threads_waiting_to_start) {
        TRACE("%d threads waiting to start...\n", threads_waiting_to_start);
        pthread_cond_broadcast(&cv);
    }

    TRACE("All threads resumed...\n");
    pthread_mutex_unlock(&lock);
}
```

(over)