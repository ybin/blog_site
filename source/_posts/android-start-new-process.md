title: android����native����
date: 2015-02-04 15:57:48
categories: android
tags: [android, process]
description: android, native process, broadcast, c, c++
---

��APP����native�����Լ���native code����intent��

<!-- more -->

### ��APP����native����

```java
try {
    Process proc = Runtime.getRuntime().exec("touch /storage/sdcard/xxx.txt");
} catch (IOException e1) {
    e1.printStackTrace();
}
```

### ��native code����intent

```c
#include <stdlib.h>

int status = -1;
status = system(am broadcast -a com.example.DEMO_ACTION --user 0");
```