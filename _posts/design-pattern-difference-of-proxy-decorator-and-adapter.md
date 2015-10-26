title: 设计模式：代理模式、装饰器模式和适配器模式之间的差异
date: 2015-05-05 14:15:06
categories: [design pattern]
tags: [design pattern]
---

proxy, decorator, adpter，三种设计模式之间的差别比较细微，尤其是proxy和decorator，
我尝试说一下自己的理解。

<!-- more -->

## 三种模式的差异概况

这三种模式，每一种都可以分为两部分，

- proxy: 原始对象(被代理对象)、代理对象
- decorator: 原始对象(被装饰对象)、装饰对象
- adaptor: 原始对象(被适配对象)、适配对象

每个模式中的原始对象都是一样的，甚至可以是同一个对象，每种模式都会创建一个新的对象，
这个新的对象当然跟原始对象有很大关联，这些关联我将其分为三部分：行为、核心内容、辅助内容。

- 行为：可以理解为接口，或者更通俗一点儿，一个函数代表一种行为
- 内容：可以理解为接口所做的工作，它可能做数值计算并返回计算结果，也可能读写文件，
    - 核心内容： 比如对于写操作，数据的write就是核心内容，
    - 辅助内容： 而write过程中打印log，发生异常时的出错提示等就属于辅助内容

```
+==============================+======+==========+==========+
|     模式中涉及的两个对象     | 行为 | 核心内容 | 辅助内容 |
+==============================+======+==========+==========+
|   origin and proxy objects   | 相同 |   相同   |   不同   |
+------------------------------+------+----------+----------+
| origin and decorated objects | 相同 |   不同   |   不同   |
+------------------------------+------+----------+----------+
|  origin and adaptor objects  | 不同 |   不同   |   不同   |
+------------------------------+------+----------+----------+
```

1. 代理对象： 与原始对象实现同一个接口，所以行为相同，代理对象直接转发原始对象
对应接口的内容，所以核心内容也是一样的，但是代理对象可以在调用原始对象接口的前后
做一些额外的工作，如写日志，增加hook等，想象一下公司的网络代理服务器统计个人流量
就清楚了，这些流量统计并非你要访问的网页的内容(核心内容)，而是一下辅助内容。
AOP编程的实现也是利用这个原理。
2. 装饰对象： 与原始对象实现同一个接口，所以行为相同，但是装饰对象在调用原始对象
对应接口前后做一些额外工作，但是与代理对象不同的是，这些工作是跟核心内容相关的，
它会增加甚至完全修改原始内容，如给网页增加滚动条，IO stream和writer, reader(bytes
转换为char，这就不只是添加了)，当然装饰对象也可以做一些诸如写日志等辅助内容。
3. 适配对象：跟原始对象实现两套不同的接口，它完成新、旧两套接口的适配工作，所以
适配对象跟原始对象的行为是不同的，它们的内容显示也应该是不同的，不过内容应该是
很相近的，如三相电转两相电，但是你不能把三相电直接接到自来水管上！

## 一个示例

通过一个将字符串写入文件的例子来演示。

首先定义接口：

```java
package com.example;

import java.io.File;

public interface StringWriteable {
    public void write(String content, File file);
}
```

下面这个接口用于adaptor做适配，

```java
package com.example;

public interface StringWriteableNew {
    public void write(String content, String filename);
}
```

下面是原始类，

```java
package com.example;

import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.io.Writer;

public class StringWriter implements StringWriteable {
    @Override
    public void write(String content, File file) {
        Writer writer = null;
        try {
            writer = new FileWriter(file);
            writer.write(content);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (writer != null) {
                try {
                    writer.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

下面是代理类，它的辅助内容即log打印输出到标准输出，

```java
package com.example;

import java.io.File;

public class StringWriterProxy implements StringWriteable {
    private StringWriteable writeable;

    public StringWriterProxy(StringWriteable writeable) {
        this.writeable = writeable;
    }

    @Override
    public void write(String content, File file) {
        System.out.println("in proxy: before write.");
        writeable.write(content, file);
        System.out.println("in proxy: after write.");
    }
}
```

然后是装饰类，

```java
package com.example;

import java.io.File;

public class StringWriterDecorator implements StringWriteable {
    private StringWriteable writeable;

    public StringWriterDecorator(StringWriteable writeable) {
        this.writeable = writeable;
    }

    @Override
    public void write(String content, File file) {
        writeable.write("this is decorated info...\n" + content, file);
    }
}
```

接着是适配类，

```java
package com.example;

import java.io.File;

public class StringWriterAdaptor implements StringWriteableNew {
    private StringWriteable writeable;

    public StringWriterAdaptor(StringWriteable w) {
        writeable = w;
    }

    @Override
    public void write(String content, String filename) {
        File file = new File(filename);
        writeable.write(content, file);
    }
}
```

最后是测试类，

```java
package com.example;

import java.io.File;

public class Test {
    public static void main(String[] args) {
        String filename = "c:/tmp_file";
        String content = "\nThis is the real content...\n";
        StringWriteable writeable;
        StringWriteableNew newWriteable;
        StringWriteable writer = new StringWriter();

        // test writer
        writeable = writer;
        writeable.write(content, new File(filename));

        // test writer proxy
        writeable = new StringWriterProxy(writer);
        writeable.write(content, new File(filename + "_proxy"));

        // test writer decorator
        writeable = new StringWriterDecorator(writer);
        writeable.write(content, new File(filename + "_decorator"));

        // test writer adaptor: StringWriteable -> StringWriteableNew
        newWriteable = new StringWriterAdaptor(writer);
        newWriteable.write(content, filename + "_adaptor");
    }
}
```

## 代理模式的细化

根据代理对象的生成方式，代理模式又分为静态代理和动态代理。

### 静态代理

静态代理就跟上面的`StringWriterProxy`一样，代理类需要手动编写，即静态编写，不再赘述。

### 动态代理

动态代理就是不用手动编写代理类，代理对象通过反射机制动态生成。代理对象的生成方式一般
有两种：JDK动态代理和cglib动态代理。

#### JDK动态代理

JDK动态代理根据接口生成代理对象，它要求必须提供接口。



#### cglib动态代理

cglib动态代理通过继承生成代理对象，因此对于final类，它无能为力。


