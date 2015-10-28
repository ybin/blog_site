title: 看看哪个程序占用了我的端口
date: 2015-10-22 10:33:09
categories: 工具
tags: [windows, port, netstat]
---

在Windows系统下，查看某个端口（如8080端口）被哪个程序占用了。

<!-- more -->

分两个步骤：

1. 查出占用该端口的进程ID
`netstat -aon | findstr 8080`
列出的最后一列即为该进程的pid。
2. 根据pid找到执行程序
`tasklist | findstr <pid>`
