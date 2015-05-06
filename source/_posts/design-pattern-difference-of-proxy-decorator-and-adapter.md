title: ���ģʽ������ģʽ��װ����ģʽ��������ģʽ֮��Ĳ���
date: 2015-05-05 14:15:06
categories: [design pattern]
tags: [design pattern]
description: design pattern, proxy, decorator, adapter, aop
---

proxy, decorator, adpter���������ģʽ֮��Ĳ��Ƚ�ϸ΢��������proxy��decorator��
�ҳ���˵һ���Լ�����⡣

<!-- more -->

## ����ģʽ�Ĳ���ſ�

������ģʽ��ÿһ�ֶ����Է�Ϊ�����֣�

- proxy: ԭʼ����(���������)���������
- decorator: ԭʼ����(��װ�ζ���)��װ�ζ���
- adaptor: ԭʼ����(���������)���������

ÿ��ģʽ�е�ԭʼ������һ���ģ�����������ͬһ������ÿ��ģʽ���ᴴ��һ���µĶ���
����µĶ���Ȼ��ԭʼ�����кܴ��������Щ�����ҽ����Ϊ�����֣���Ϊ���������ݡ��������ݡ�

- ��Ϊ���������Ϊ�ӿڣ����߸�ͨ��һ�����һ����������һ����Ϊ
- ���ݣ��������Ϊ�ӿ������Ĺ���������������ֵ���㲢���ؼ�������Ҳ���ܶ�д�ļ���
    - �������ݣ� �������д���������ݵ�write���Ǻ������ݣ�
    - �������ݣ� ��write�����д�ӡlog�������쳣ʱ�ĳ�����ʾ�Ⱦ����ڸ�������

```
+==============================+======+==========+==========+
|     ģʽ���漰����������     | ��Ϊ | �������� | �������� |
+==============================+======+==========+==========+
|   origin and proxy objects   | ��ͬ |   ��ͬ   |   ��ͬ   |
+------------------------------+------+----------+----------+
| origin and decorated objects | ��ͬ |   ��ͬ   |   ��ͬ   |
+------------------------------+------+----------+----------+
|  origin and adaptor objects  | ��ͬ |   ��ͬ   |   ��ͬ   |
+------------------------------+------+----------+----------+
```

1. ������� ��ԭʼ����ʵ��ͬһ���ӿڣ�������Ϊ��ͬ���������ֱ��ת��ԭʼ����
��Ӧ�ӿڵ����ݣ����Ժ�������Ҳ��һ���ģ����Ǵ����������ڵ���ԭʼ����ӿڵ�ǰ��
��һЩ����Ĺ�������д��־������hook�ȣ�����һ�¹�˾��������������ͳ�Ƹ�������
������ˣ���Щ����ͳ�Ʋ�����Ҫ���ʵ���ҳ������(��������)������һ�¸������ݡ�
AOP��̵�ʵ��Ҳ���������ԭ��
2. װ�ζ��� ��ԭʼ����ʵ��ͬһ���ӿڣ�������Ϊ��ͬ������װ�ζ����ڵ���ԭʼ����
��Ӧ�ӿ�ǰ����һЩ���⹤����������������ͬ���ǣ���Щ�����Ǹ�����������صģ�
��������������ȫ�޸�ԭʼ���ݣ������ҳ���ӹ�������IO stream��writer, reader(bytes
ת��Ϊchar����Ͳ�ֻ�������)����Ȼװ�ζ���Ҳ������һЩ����д��־�ȸ������ݡ�
3. ������󣺸�ԭʼ����ʵ�����ײ�ͬ�Ľӿڣ�������¡������׽ӿڵ����乤��������
��������ԭʼ�������Ϊ�ǲ�ͬ�ģ����ǵ�������ʾҲӦ���ǲ�ͬ�ģ���������Ӧ����
������ģ��������ת����磬�����㲻�ܰ������ֱ�ӽӵ�����ˮ���ϣ�

## һ��ʾ��

ͨ��һ�����ַ���д���ļ�����������ʾ��

���ȶ���ӿڣ�

```java
package com.example;

import java.io.File;

public interface StringWriteable {
    public void write(String content, File file);
}
```

��������ӿ�����adaptor�����䣬

```java
package com.example;

public interface StringWriteableNew {
    public void write(String content, String filename);
}
```

������ԭʼ�࣬

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

�����Ǵ����࣬���ĸ������ݼ�log��ӡ�������׼�����

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

Ȼ����װ���࣬

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

�����������࣬

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

����ǲ����࣬

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

## ����ģʽ��ϸ��

���ݴ����������ɷ�ʽ������ģʽ�ַ�Ϊ��̬����Ͷ�̬����

### ��̬����

��̬����͸������`StringWriterProxy`һ������������Ҫ�ֶ���д������̬��д������׸����

### ��̬����

��̬������ǲ����ֶ���д�����࣬�������ͨ��������ƶ�̬���ɡ������������ɷ�ʽһ��
�����֣�JDK��̬�����cglib��̬����

#### JDK��̬����

JDK��̬������ݽӿ����ɴ��������Ҫ������ṩ�ӿڡ�

#### cglib��̬����

cglib��̬����ͨ���̳����ɴ��������˶���final�࣬������Ϊ����
