title: 安装JamVM
date: 2014-11-18 14:43:15
categories: jvm
tags: [JamVM, JVM]
---

[JamVM](http://jamvm.sourceforge.net) 没有二进制版本，只能从[源码](http://sourceforge.net/projects/jamvm/files/jamvm/JamVM%202.0.0/) 编译安装。

JamVM只是一个虚拟机，它需要搭载一个Java class library才能正常运行，可以使用
GNU classpath，也可以使用OpenJDK，这里我们使用GNU classpath。

<!--more-->

#### 编译安装GNU classpath

GNU classpath是一个Java class library，同样需要从源码开始编译。

- 下载[源码](ftp://ftp.gnu.org/gnu/classpath/classpath-0.99.tar.gz)
- 编译
```bash
./configure --prefix=/tmp/classpath --disable-gtk-peer --disable-gconf-peer --disable-plugin
```
- 安装
```bash
# -i 忽略warning和一些error
make -i
make -i install
```

编译需要Java环境，需要系统中预先安装Java环境。另外还会用到`ANTLR`，如果系统中
没有安装的话，可以按照如下方式安装，[参考官网](https://theantlrguy.atlassian.net/wiki/display/ANTLR4/Getting+Started+with+ANTLR+v4)：

- 下载antlr
- 添加jar文件至CLASSPATH
- 添加alias
- 以上是官网的安装步骤，但是classpath的configure仍然无法识别antlr，所以要
```bash
export ANTLR_JAR=/your/path/to/antlr jar file
```

#### 安装JamVM

同样是从[源码](http://sourceforge.net/projects/jamvm/files/jamvm/JamVM%202.0.0/)开始编译安装。

1. 下载[源码](http://sourceforge.net/projects/jamvm/files/jamvm/JamVM%202.0.0/)
2. 编译： `./configure --prefix=/tmp/jamvm --with-classpath-install-dir=/tmp/classpath`
3. 安装： `make && make install`
4. 设置环境变量： `export PATH=/tmp/jamvm/bin:$PATH`

(over)