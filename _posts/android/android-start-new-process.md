title: android启动native进程
date: 2015-02-04 15:57:48
categories: android
tags: [android, process]
---

从APP启动native进程以及从native code发送intent。

<!-- more -->

### 从APP启动native进程

```java
try {
    Process proc = Runtime.getRuntime().exec("touch /storage/sdcard/xxx.txt");
} catch (IOException e1) {
    e1.printStackTrace();
}
```

### 从native code发送intent

```c
#include <stdlib.h>

int status = -1;
status = system(am broadcast -a com.example.DEMO_ACTION --user 0");
```