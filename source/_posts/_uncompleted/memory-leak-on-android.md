title: Android里的内存泄露
date: 2015-03-11 08:49:10
categories: android
tags: [android, memory leak]
---

### static field

static field存储在Class object里面，所以，被static field引用的对象不会被gc，
除非Class object主动释放该

### inner class and anounymous inner class