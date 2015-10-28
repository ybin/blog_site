title: Bash Tips
date: 2015-03-16 15:29:16
categories: 编程
tags: [bash, tip]
---

Bash tips.

<!-- more -->

### 关于“当前工作路径”

在`调用`、`引入`其他shell脚本时，当前工作路径有什么变化？

答案是：除非执行`cd`，否则没有变化。

实验环境：
```bash
+-build.sh
+-a
| +-b
| | +-c
| | | +-main.sh
| | | +-util.sh
```