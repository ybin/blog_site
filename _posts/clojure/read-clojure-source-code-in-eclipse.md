title: 使用Eclipse阅读clojure源码(Java部分)
date: 2015-01-20 12:30:02
categories: clojure
tags: [clojure, eclipse, java]
---

如何使用eclipse阅读[clojure][clojure]源码的Java部分。

<!-- more -->

1. 下载[clojure源码][clojure]
2. 在eclipse中创建Java Project，名为clojure
3. 依次打开：clojure项目的Properties窗口 -> Java Build Path -> Source
4. 打开Link Source窗口，Linked folder location填clojure源码的src\jvm路径，
Folder name自定义，一般为src，如果已经有src目录就先删除它
5. 完成

Java代码可以正常编译，但是如果打包为.jar文件则需要src/clj/目录里的内容，这需要
ant或者mvn才行。
