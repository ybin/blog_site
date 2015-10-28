title: git push的默认行为
date: 2015-05-21 09:19:38
categories: git
tags: [git]
---

`git push`的默认行为介绍。

<!-- more -->


```c
/*
+----------+--------------+------------------------------------------+
|          | local branch | remote branch                            |
+==========+==============+==========================================+
| nothing  | none         | none                                     |
+----------+--------------+------------------------------------------+
| current  | current      | same name (create if necessary)          |
+----------+--------------+------------------------------------------+
| upstream | current      | upstream                                 |
+----------+--------------+------------------------------------------+
| simple   | current      | upstream && same name                    |
|                         | (like current when push to other remotes)|
+----------+--------------+------------------------------------------+
| matching | all          | same name                                |
+----------+--------------+------------------------------------------+
*/
```

重点区别`current`, `upstream`, `matching`，

- `current`: 把当前分支推送到远程同名分支，如果远程同名分支不存在就自动创建
- `upstream`: 把当前分支推送到远程跟踪分支(upstream)，远程跟踪分支必须存在，但是不必跟当前分支同名
- `simple`: 把当前分支推送到远程跟踪分支(upstream)，远程跟踪分支必须存在，且与当前分支同名

### 跟踪分支(upstream)是什么意思？

看config文件，分支`g`的跟踪分支就是远程的`b`分支
```
[branch "g"]
	remote = origin
	merge = refs/heads/b
```

### 如何设置跟踪分支(upstream)？

自动设置远程分支b为本地分支g的跟踪分支：

1. `git checkout -b g origin/b`
2. `git branch g; git branch --track origin/b g;`

### 使用推荐

`simple`是严格的(除了`nothing`)，适合初学者使用，在central workflow里，它类似于`upstream`，
但是更严格(要求本地、远程分支同名)，在non-central workflow里，如同时推送代码到多个remotes，
此时它等同于`current`，推送到同名分支，如果没有就创建。

`upstream`是最适合日常使用的，它比`simple`宽松，不要求同名，如可以这样，其他跟`simple`一样，

```
git branch --set-upstream-to=eric/master eric_master
git branch --set-upstream-to=john/master john_master
git branch --set-upstream-to=john/bugfix john_bugfix
```

(over)