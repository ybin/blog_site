title: clojure术语之：sequence, seq, sequential and persistent collections
date: 2015-02-05 09:32:32
categories: clojure
tags: [clojure, sequence, seq, sequential, collection]
---

clojure中的sequence, seq, sequential以及persistent collections之间的关系。

<!-- more -->

clojure提供了一些内置的组合数据类型(composite data type，也叫clojure collections)，
如list, vector, map, set，它们是与数字、字符串等简单数据类型相对的。

### sequential

clojure collection(区别于java collection)根据**是否保持次序**又分为两类:

1. sequential
2. not sequential

什么叫做**是否保持次序**呢？保持次序意味着增加或删除元素之后，原来的元素保持次序
不变，如一个vector: `[1 2 3 4 5]`，现在删除`4`，则剩下的一定是`[1 2 3 5]`，剩下
的元素仍然保持原有的次序，相比之下一个set: `#{1 2 3 4 4}`，它有一定的顺序，只是
这个顺序是clojure内部使用的，用户不应该基于这个次序来操作，现在删除`4`，则剩下的
元素无法保证其次序不会改变。

Sequential是一个dumy interface，它没有任何method，只是实现该接口的对象必须在实现
时自己保证元素次序。

### seq

java collection有好几种，map, set, list，他们各自都有自己独特的操作方法，其他语言
如python也是如此，其list, turple等都有自己独特的操作方法。那么clojure呢，clojure
collection也有好几种：list, vector, map, set，它们是不是也要各自搞出一套操作方法
来呢，非也！

clojure为所有的clojure collection提供一套统一的操作方法，即ISeq，也就是说任何collection
都可以转换为ISeq对象，这是通过Seqable接口保证的，

```java
public interface Seqable {
    ISeq seq();
}

public interface IPersistentCollection extends Seqable {
    /* ... */
}
```

ISeq为collection提供了另一个**视角**，于是调用任何collection对象的`seq()`接口就会
得到一个ISeq对象，然后就可以操作这个ISeq对象来操作collection。

```java
public interface ISeq extends IPersistentCollection {

    Object first();

    ISeq next();

    ISeq more();

    ISeq cons(Object o);
}
```

这样看来`seq`是一个接口，其实它还是一个函数，clojure提供一个`seq`函数，它将任何collection
以及其他一些对象(如String)转换为ISeq对象。

### sequence

ISeq对象就叫做sequence，有了seq接口和seq函数的支持，我们只需操作sequence就可以了，于是clojure
提供一套强大的函数库，叫做sequence函数库，来实现对sequence的各种操作，`map`, `reduce`, `filter`
等等，所有的clojure collection都可以通过这个函数库来进行操作。

(over)