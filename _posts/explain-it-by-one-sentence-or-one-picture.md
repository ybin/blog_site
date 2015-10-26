title: 一句话系列
date: 2015-03-06 15:22:01
categories: [编程]
tags: [一句话]
---

一句话或一幅图解释原理。

<!-- more -->

### overriding vs overloading

![overriding vs overloading](/res/img/overloading.vs.overriding.png)

图片[来源][override-vs-overload]。

### 依赖注入(DI, Dependency Injection)

```java
// 非依赖注入
public class Human {
    //...
    Father father;
    //...
    public Human() {
        father = new Father();
    }
}

// 依赖注入
//  - 非自己主动初始化依赖，而通过外部来传入依赖的方式，我们就称为依赖注入。
public class Human {
    //...
    Father father;
    //...
    public Human(Father father) {
        this.father = father;
    }
}
```

### 在Java中，什么时候可以省略数组的大小？

一句话：声明的时候可以忽略，但是创建的时候不能省略，因为数组的大小是**不可变**的。

```java
int[] a;    // ok, declare.
int[][] a0; // ok, declare.
int[][] a1 = new int[3][]; // 第一个尺寸不能省略，第二个则可以。
a1[0][0] = 1; // fail. null pointer.
int[][][] a2 = new int[3][][]; // ok, 同上。
```

不论是多少维的数组，创建时第一个维度信息都不能省略，其他维度信息则可以省略，为什么？
答：
一个N维数组其实可以就是：一个一维数组，其元素为N-1维的数组。所以创建N维数组的时候，
第一个维度信息不能省略，但是试图访问具体元素时就会出现空指针，因为并未开辟内存空间。


### 如何获取数组和字符串的长度信息？为什么？

```java
int[] arr = new int[3];
System.out.println(arr.length);
String str = "abc";
System.out.println(str.length());
```

因为数组对象有一个`length`属性，而且是**不可变的**，而字符串使用字符数组保存数据，
字符数组本身就有长度信息，所以String没有`length`属性，故提供`length()`方法。

### Java中，equals()和hashCode()的关系

两个对象，
- 如果equals()为true，则hashCode()返回值一定相等
- 如果hashCode()返回值相等，equals()**不一定**为true

在HashMap中，查找对象时，首先根据其hash code进行快速定位，如果找到了，在根据
`equals()`值判断是否相等，如果相等则找到了，否则就返回Null。

所以，
- 如果两个对象equal，但是hash code不相等，则要么第一步就找不到，要么第二步
equals()为false，总之就是找不到该对象。
- 如果两个对象hash code相等但是equals()为true，则会把不同对象hash到同一个“槽”里面，
但是这种情况是运行的。


### Java中的各种initializer

```java
public class Foo {
    // instance variable initializer， 等价于instance initializer
    String s = "abc";
    
    // constructor
    public Foo () {
        System.out.println("constructor called") ;
    }
    
    // static initializer
    static {
        System.out.println("static initializer called") ;
    }
    
    // instance initializer
    {
        System.out.println("instance initializer called") ;
    }

    public static void main(String [] args) {
        new Foo();
        new Foo();
    }
}
```

输出，

```bash
static initializer called
instance initializer called
constructor called
instance initializer called # 第二个对象的输出
constructor called
```

instance initializer不常用，比较可以在构造函数里面做这些事情就好了，
但是有一种情况：匿名内部类，可以使用instance initializer，并将它没有自定义的
构造函数。

```java
HashMap<String, String> m = new HashMap<String, String>() {
    {
        // instance initializer
        put("a", "1");
        put("b", "2");
    }
};
```

### Java exceptions

一句话：
- cheched exception表示program之外发生的异常，program对此无能为力，需要programmer介入处理，如file not found；
- unchecked exception表示program本身发生的异常，是program自身的逻辑问题，如除数为0。

Exception hirachy:

![Java Exception Hirachy](/res/img/JavaExceptions.jpeg)

红色表示checked exception，蓝色表示unchecked exception。

compiler会关注checked exception，这类exception必须处理，否则编译不过。

所有Exception均可以捕获，

```java
    int a = 0;
    a = 1/0; // 编译没有问题，运行错误。下面的代码，编译、运行均正常。
    
    try {
        a = 1/0;
    } catch (RuntimeException e1) {
    }
    System.out.println(a);
    
    C c = null;
    try {
        c.c = 0;
    } catch (RuntimeException e) {
        c = new C();
        c.c = 1;
    }
    System.out.println(c.c);
```





代码[来源][dependency-injection]。

[override-vs-overload](http://www.programcreek.com/2009/02/overriding-and-overloading-in-java-with-examples/)
[dependency-injection](http://www.codekk.com/open-source-project-analysis/detail/Android/%E6%89%94%E7%89%A9%E7%BA%BF/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)
