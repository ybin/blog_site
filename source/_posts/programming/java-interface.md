title: Java Interface
date: 2015-01-26 11:09:11
categories: 编程
tags: [java]
---

Java Interface中的方法和属性都是如何修饰的，即可以使用什么修饰符？

<!-- more -->

在Java中interface中可以有method和field，但是它们的修饰符有严格要求，即

1. interface中的method**只能**用`public abstract`来修饰
2. interface中的field**只能**用`public static final`来修饰

当然，写代码的时候你可以选择写完整的修饰符，也可以只写一部分修饰符，
甚至可以完全不用写任何修饰符，不论哪种写法，都不会影响最终的结果。

例如，

```java
public interface IMyInterface {
    void open();
    public void open();
    public abstract open();
    // 以上写法是等价的
    
    // 以下写法也是等价的
    int fileNum = 0;
    public int fileNum = 0;
    public static int fileNum = 0;
    public static final int fileNum = 0;
    final int fileNum = 0;
    // ...
}
```

(over)