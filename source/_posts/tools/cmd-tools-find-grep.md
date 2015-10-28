title: 命令行工具：find
date: 2015-05-14 14:15:48
categories: [工具]
tags: [工具, find]
---

find工具的简单用法。

<!-- more -->

# 统计代码行数

```bash
find . -type f -name "*.java" -or -name "*.xml" | xargs cat | wc -l
# for GoW on Windows
find . -type f -name "*.java" -or -name "*.xml" | tr \r\n ' ' | xargs cat | wc -l

# 代码说明：
-type f # 文件类型只关注file
-name "*.java" -or -name "*.xml" # 只关注java和xml文件
xargs cat # 打印出文件内容
wc -l # 统计行数
tr \r\n ' ' # 将\r\n转换为空格，否则xargs cat会无法正常分割文件名，导致找不到文件
```

# 递归搜索整个目录

```bash
find . -type f -name "*.java" -exec grep -i "onCreate" {} \;

# 代码说明
-type f -name "*.java" # 找到所有的java文件
-exec grep -inH "onCreate" {} \; # 每找到一个就执行grep命令，{}代表找到的文件的文件名， \; 防止分号被转意
                               # Windows上不需要分号转意
                               # -i: 忽略大小写， -n: 输出行号， -H: 输出文件名
```