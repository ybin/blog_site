title: Android C++代码中的强指针和弱指针
date: 2015-04-07 16:10:36
categories: android
tags: [android]
---

Android源码中，C++层面中的强指针和弱指针实现原理。

<!-- more -->

说明：该部分代码已经移植到Linux系统上，具体代码以下面的示例代码均可以在我的
[github repo][repo]中下载到。

### 为何要用强、弱指针

C/C++中内存管理一直是让程序员头疼的地方，申请一段内存要记得释放，否则就会内存泄漏，
它不像Java能够自动回收。Android框架部分大量使用C++实现，内存管理尤为重要，而最简单
的自动内存管理方法就是`计数`，每个对象都有一个计数器，当增加引用时计数器增加，当
减少引用时计数器减小，当计数器减为0时，对象自动释放。强、弱指针的根本原理也是这样的。

但是简单的计数方法有个缺陷，那就是`循环引用`，当两个对象循环引用时，谁都无法释放，
从而导致内存泄漏，Android引入弱指针的概念，从而解决了循环引用的问题，而且弱引用在
实现对象`缓存`方面也是非常方便的，这两个场景是弱引用的主要服务对象。弱引用的存在主要
解决了这样的问题：即使存在多个弱引用，但是对象仍然可以正常释放。

### 强、弱指针的实现原理

具体如何实现呢，一般的实现方法是给每个对象添加一个计数器，其实就是一个`int`字段，
但是考虑到弱指针需要弱引用计数器，Android并不是简单的添加一个计数器，而是为每个对象
创建一个额外的弱引用对象，叫做weakref_type，它里面包括强引用计数器、弱引用计数器、
base对象指针以及对象生命周期标志位，base对象和其弱引用对象在内存中的示意图如下，

```c
/*
  RefBase object             weakref_type object

+-----------------+     +---> +-------------+
|      mRefs      |-----+     |   mStrong   |
+-----------------+           +-------------+
|  The real       |           |   mWeak     |
|  content of     |           +-------------+
|  object, e.g.   |           |   mBase     |
|  CameraService  |           +-------------+
|  object         |           |   mFlag     |
+-----------------+           +-------------+

*/
```

两个对象总是成对儿存在的，它们总是**共进退**。

RefBase对象只有一个字段: `mRefs`，它是一个指向计数对象(weakref_type)对象的指针。
RefBase类中定义了强引用相关的操作，比如`incStrong()`、`decStrong()`等，而weakref_type
类中定义了弱引用相关的操作，比如`incWeak()`、`decWeak()`、`promote()`等操作，如图示，

![refbase diagram](/res/img/refbase-diagram.png)

对象的布局定义好了，我们还需要操作对象的手段，以达到`**自动**`释放对象的目的，为此
android创建了`template <typename T> class sp`和`template <typename T> class sp`两个类，
`sp`对象只有一个字段，那就是`RefBase`对象的指针，`wp`对象中有两个字段，分别是指向
`RefBase`和`weakref_type`对象的指针，

![sp and wp diagram](/res/img/sp-wp-diagram.png)

然后这两个类中重载了一堆操作符，如赋值操作，解引用操作，比较运算符等等，以至于我们
可以像使用base object一样使用`sp`和`wp`对象。
然后就是关键的构造函数和析构函数，所谓的自动操作均在sp和wp的构造函数中实现，

- sp在构造函数中增加强指针计数，在析构函数中减小强指针计数，
- wp在构造函数中增加弱指针计数，在析构函数中减小弱指针计数。

而sp, wp对象是在栈(stack)上创建的(是的，这是使用强弱指针的关键)，所以sp, wp对象在
跳出作用域之后会**自动析构**，这是整个系统实现的关键所在。

### 源码分析

代码主要分为三部分：

1. 强指针的相关操作，在RefBase类中
2. 弱指针的相关操作，在RefBase::weakref_type类中
3. sp, wp实现自动化操作

原则上来讲，强指针只修改强引用计数，弱指针只修改弱引用计数，它们只关注自己
的计数器而不能染指其他计数器，但是强弱指针之间又有关联，那就是“强引用的增减
一定会导致弱引用的增减”。

#### 强指针的相关操作

强指针的操作主要是：`incStrong()`和`decStrong()`，它们实现强引用计数的增减。
`extendObjectLifetime()`是protected的，它可以修改`mFlag`标志位，该标志位用来
控制对象的生命周期，当前有两种生命周期：
1. OBJECT_LIFETIME_STRONG: 对象生命周期由强指针控制，即强引用为0时释放对象内存
2. OBJECT_LIFETIME_WEAK: 对象生命周期由弱指针控制，即弱引用为0时释放对象内存
另外，`onFirstRef()`, `onLastStrongRef()`，顾名思义，在第一次和最后一次引用时
调用，子类可以覆盖这些方法，有点儿“构造”、“析构”的意思，`onIncStrongAttempted()`
函数是弱引用promote为强引用时用到的，子类可以覆盖该方法，用于表明自己是否愿意
由弱转强，即给智能指针系统一个提示从而表明自己的态度，onXXX()这几个函数可以
理解为智能指针系统留给之类的接口，从而让子类一定程度上参与到指针的管理中来。

下面单独分析`incStrong()`，`decStrong()`和`~RefBase()`，这几个函数是强指针实现
的关键。

```c
void RefBase::incStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    // 强引用的增减一定会导致弱引用的增减，这是强弱指针之间的关联关系
    refs->incWeak(id);
    // 增减强引用计数
    const int32_t c = android_atomic_inc(&refs->mStrong);
    // 如果不是第一次增减强引用计数，那就没有别的可做了，增减计数就完了
    if (c != INITIAL_STRONG_VALUE)  {
        return;
    }
    // 第一次增减强引用计数，即第一次创建强引用，强引用计数从"初始值 + 1"改为"1"
    // 初始值之所以不是"0"，就是为了跟"引用计数减为0时释放对象"这个场景区分开
    android_atomic_add(-INITIAL_STRONG_VALUE, &refs->mStrong);
    // 第一次创建强引用，调用子类接口，完成初始化工作
    refs->mBase->onFirstRef();
}
```

```c
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    // 强引用计数减小
    const int32_t c = android_atomic_dec(&refs->mStrong);
    // 如果强引用计数减为0
    if (c == 1) {
        // 调用子类接口，完成清理工作
        refs->mBase->onLastStrongRef(id);
        // 如果对象的lifetime是strong，就释放对象内存
        if ((refs->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this;
        }
    }
    // 弱引用计数减小
    refs->decWeak(id);
}
```

```c
RefBase::RefBase()
    : mRefs(new weakref_impl(this)) { }

RefBase::~RefBase() {
    if (mRefs->mStrong == INITIAL_STRONG_VALUE) {
        // we never acquired a strong (and/or weak) reference on this object.
        delete mRefs;
    } else {
        // life-time of this object is extended to WEAK or FOREVER, in
        // which case weakref_impl doesn't out-live the object and we
        // can free it now.
        if ((mRefs->mFlags & OBJECT_LIFETIME_MASK) != OBJECT_LIFETIME_STRONG) {
            // It's possible that the weak count is not 0 if the object
            // re-acquired a weak reference in its destructor
            if (mRefs->mWeak == 0) {
                delete mRefs;
            }
        }
    }
}
```

总结：
1. `incStrong()`: inc weak counter => inc strong counter, 如果是第一个强指针，调用子类接口进行初始化工作
2. `decStrong()`: dec strong counter => dec weak counter, 如果是最后一个强指针，调用子类接口进行清理工作
3. `~RefBase()` : 释放计数器对象，计数器对象是RefBase对象的一部分，所以在析构时要记得释放资源，这要跟decWeak()
配合进行，否则多次delete是会出问题的。

#### 弱指针的相关操作

弱引用的操作集中在：`incWeak()`, `decWeak()`, `attemptIncStrong()`三个操作上。
弱引用存在的目的是：即使存在多个弱引用，对象仍然可以正常释放。弱引用对象(wp)不能直接操作对象，因为它没有
覆盖解引用操作符和指针操作符(->)，要想操作对象，只能先将弱引用对象(wp)提升为强引用对象(sp)，然后才能进行
操作，而提升要通过`attemptIncStrong()`才能进行。

```c
void RefBase::weakref_type::incWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    // 简单的inc即可
    const int32_t c = android_atomic_inc(&impl->mWeak);
}
```

```c
void RefBase::weakref_type::decWeak(const void* id)
{
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    // 减小弱引用计数
    const int32_t c = android_atomic_dec(&impl->mWeak);
    if (c != 1) return;

    // 如果弱引用计数减为0
    if ((impl->mFlags&OBJECT_LIFETIME_WEAK) == OBJECT_LIFETIME_STRONG) {
        // This is the regular lifetime case. The object is destroyed
        // when the last strong reference goes away. Since weakref_impl
        // outlive the object, it is not destroyed in the dtor, and
        // we'll have to do it here.
        // 如果对象生命周期是强引用控制
        if (impl->mStrong == INITIAL_STRONG_VALUE) {
            // 但是有可能压根儿没有创建过强引用对象(sp)，那么我们在这里释放对象
            delete impl->mBase;
        } else {
            // 否则强引用对象(sp)肯定已经释放掉对象了，因为弱引用计数一定大于强引用计数，
            // 那么我们只需要删掉计数器对象即可
            delete impl;
        }
    } else {
        // less common case: lifetime is OBJECT_LIFETIME_{WEAK|FOREVER}
        // 如果对象生命周期是弱引用控制或者没有任何控制
        // 调用子类接口完成清理工作
        impl->mBase->onLastWeakRef(id);
        if ((impl->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_WEAK) {
            // 如果对象生命周期是弱引用控制，那么我们释放对象，
            //? 但是，计数对象由谁来释放呢？答案是：计数器对象在RefBase的析构函数里面释放
            delete impl->mBase;
        }
    }
}
```

流程图如下，

![decWeak流程图](/res/img/decWeak-diagram.png)

```c
bool RefBase::weakref_type::attemptIncStrong(const void* id) {
    incWeak(id);
    weakref_impl* const impl = static_cast<weakref_impl*>(this);
    
    // 根据强引用计数的大小来分别对待：
    int32_t curCount = impl->mStrong;
    
    // 1. 强引用计数大于0且不为初始值，即存在其他强引用，
    //    此时直接增加强引用计数，但是注意与其他强引用作并发处理
    while (curCount > 0 && curCount != INITIAL_STRONG_VALUE) {
        if (android_atomic_cmpxchg(curCount, curCount+1, &impl->mStrong) == 0) {
            break;
        }
        curCount = impl->mStrong;
    }
    // 2. 不存在其他强引用，即强引用计数为初始值或者为0(其他强引用均已释放了)，
    //    此时要区别对待，
    if (curCount <= 0 || curCount == INITIAL_STRONG_VALUE) {
        bool allow;
        if (curCount == INITIAL_STRONG_VALUE) {
            // 2.1 强引用计数为初始值，即从没有创建过强引用，此时要求生命周期由
            //     强引用控制**或者**RefBase对象或其子类对象允许inc strong。
            //     强引用计数为初始值说明base object一定是存在的。
            allow = (impl->mFlags&OBJECT_LIFETIME_WEAK) != OBJECT_LIFETIME_WEAK
                  || impl->mBase->onIncStrongAttempted(FIRST_INC_STRONG, id);
        } else {
            // 2.2 强引用计数为0，此时要求生命周期是弱引用控制**并且**RefBase对象
            //     或其子类对象允许inc strong，这里生命周期必须是弱引用控制的，
            //     否则如果由强引用控制，那么base object一定已经被干掉了，肯定无法提升。
            allow = (impl->mFlags&OBJECT_LIFETIME_WEAK) == OBJECT_LIFETIME_WEAK
                  && impl->mBase->onIncStrongAttempted(FIRST_INC_STRONG, id);
        }
        if (!allow) {
            // 如果无法inc strong，恢复weak counter，然后返回false
            decWeak(id);
            return false;
        }
        // 允许inc strong，所以inc it
        curCount = android_atomic_inc(&impl->mStrong);

        // If the strong reference count has already been incremented by
        // someone else, the implementor of onIncStrongAttempted() is holding
        // an unneeded reference.  So call onLastStrongRef() here to remove it.
        // (No, this is not pretty.)  Note that we MUST NOT do this if we
        // are in fact acquiring the first reference.
        if (curCount > 0 && curCount < INITIAL_STRONG_VALUE) {
            impl->mBase->onLastStrongRef(id);
        }
    }
    // 如果是首次inc strong，记得调用子类接口进行初始化
    if (curCount == INITIAL_STRONG_VALUE) {
        android_atomic_add(-INITIAL_STRONG_VALUE, &impl->mStrong);
        impl->mBase->onFirstRef();
    }
    // 最后，提升成功
    return true;
}
```

流程图如下，注意这个流程图不是完全按照代码顺序来的，但是逻辑上是等价的，

![attemptIncStrong 流程图](/res/img/attemptIncStrong-diagram.png)

总结：
弱指针的相关操作是比较复杂的，因为他要考虑强指针的一些情况，还要进行promote由弱转强，
这几个函数的逻辑是比较复杂的。promote时一个先决条件就是：RefBase对象是存在的，如果它
不存在，那么wp的promote压根儿不会进行。

#### sp, wp的相关操作

1. sp只会用到两个函数：`incStrong()`, `decStrong()`，分别在sp的构造(或赋值)、析构时用到
2. wp除了用到`incWeak()`, `decWeak()`之外，在其promote时还会用到`onIncStrongAttempted()`，
分别在wp的构造(或赋值)、析构以及promote时用到。


### 强、弱指针的使用方法

用法，是的，原理逻辑复杂，但是用法却很简单。

```cpp
class A : public RefBase {
public: 
    A() {
        extendObjectLifetime(OBJECT_LIFETIME_STRONG); // 
    }
    ~A() {
        printf("delete A\n");
    } 
};

static void print(char* name, RefBase* ref) {
    printf("%s: strong ref count: %d\n", name, ref->getStrongCount()); 
    printf("%s: weak ref count: %d\n", name, ref->getWeakRefs()->getWeakCount());
}

int main()
{ 
    sp<A> spa = new A();
    print("A", spa.get());
    /* 输出：
        A: strong ref count: 1
        A: weak ref count: 1
        delete strong A
     */
    
    return 0;
}
```


[repo]: https://github.com/ybin/AndroidDemo/tree/master/refbase