title: 一句话系列
date: 2015-03-06 15:22:01
categories: [编程]
tags: [一句话]
description: 一句话, 一幅图, 一段码，编程
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

代码[来源][dependency-injection]。

[override-vs-overload](http://www.programcreek.com/2009/02/overriding-and-overloading-in-java-with-examples/)
[dependency-injection](http://www.codekk.com/open-source-project-analysis/detail/Android/%E6%89%94%E7%89%A9%E7%BA%BF/%E5%85%AC%E5%85%B1%E6%8A%80%E6%9C%AF%E7%82%B9%E4%B9%8B%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5)
