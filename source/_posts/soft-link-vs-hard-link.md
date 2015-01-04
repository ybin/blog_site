title: 彻底理解“软链接”和“硬链接”
date: 2015-01-04 14:10:37
categories: linux/unix
tags: [soft-link, hard-link]
---

软链接(soft link)到底“软”在哪里，硬链接(hard link)到底“硬”在何处，本文一一道来。

<!-- more -->

### 背景知识：文件系统
linux中，在文件系统的视角来看，文件其实是inode，一个文件对应于一个inode，在inode
中保存有文件的元信息(metadata)，比如文件大小，读写权限，设备号，操作接口(read, write etc.)
等，以及文件的实际数据，图示一下，

![ext2-inode](/res/img/ext2-inode.jpg)

图中的"Infos"就是文件的元信息，而"blocks"保存的就是文件的实际数据。

目录，也是一个文件，只不过它的实际数据就是一些`(filename, inode number)`而已，
这些“文件名-inode号”叫做`dirent`，

```c
struct dirent {
    uint inum;
    char name[DIRSIZE];
}
```

### 图解软、硬连接
那软、硬连接是如何存在于文件系统中的呢，他们到底有何区别呢？下面是完整的图示，
注意，这些图示来源于[这里][link]。

```c
/*
0. inode在磁盘上大致是这个样子的：
                        .---------------> ! data ! ! data ! etc
                       /                  +------+ !------+
 ! permbits, etc ! data addresses !
 +------------inode---------------+

1. 加上dirent信息，
                  .--------------> ! permbits, etc ! data addresses !
                 /                 +-------------inode--------------+
 ! filename ! inode # !
 +--------dirent------+

2. 这个就是硬链接(hard link)，即多个dirent连接到同一个inode，多个dirent互为“别名”
 ! filename ! inode # !
 +--------------------+
                 \
                  >--------------> ! permbits, etc ! addresses !
                 /                 +---------inode-------------+
 ! othername ! inode # !
 +---------------------+

3. 这个就是软连接(soft link)
      ! filename ! inode # !
      +--------------------+
                      \
                       '-------> ! permbits, etc ! addresses !
                                 +---------inode-------------+
                                                    /
                                                   /
                                                  /
  .----------------------------------------------'
 ( 
  '-->  !"/path/to/some/other/file"! 
        +---------data-------------+
                /                      }
  .~ ~ ~ ~ ~ ~ ~                       }-- (redirected at open() time)
 (                                     }
  '~~> ! filename ! inode # !
       +--------------------+
                       \
                        '------------> ! permbits, etc ! addresses !
                                       +---------inode-------------+
                                                          /
                                                         /
   .----------------------------------------------------'
  (
   '->  ! data !  ! data ! etc.
        +------+  +------+ 

*/
```

硬链接比较好理解，多个dirent互为别名，他们拥有同一个inode，inode里面会记录link数量，
删除文件时，只有当link减为0的时候才会真正删除数据，否则只是减少link数而已。

而软连接则是一个“独立”的文件，它有自己的dirent，有自己的inode，只是它的inode中并非
文件数据而是另一个文件的路径而已，当然inode里面也有相关的类型信息，也就是一些标志位，
表明这个inode是一个symbolic link，这样在open()的时候操作系统才会根据其内容进行重定位。
这涉及到软连接的存储方式，早期软连接的实现是采用直接分配磁盘空间的方法，这种机制与
普通文件一致，也就是上图所图示的方式。但是这种方式有些缓慢而且浪费磁盘空间，所以又
发明了一种名为**快速符号连接**的存储方式，它会将文本形式的链接存同文件元信息存储在
一起，都放在inode里面。

### 一个实例
在当前目录下创建三个文件：`orig`, `hard`, `soft`

```bash
$ touch orig      # origin file
$ ln orig hard    # hard link
$ ln -s orig soft # soft link
```

然后我们来读取它们的类型，

```c
#include <stdio.h>
#include <sys/stat.h>

int main() {
    char *p[] = { "./orig", "./hard", "./soft" };
    struct stat buf;
    int i;
    for(i = 0; i < 3; i++) {
        // read file stat info
        lstat(p[i], &buf);
        switch(buf.st_mode & S_IFMT) {
            case S_IFLNK:
                printf("%s is a symbolic link.\n", p[i]);
                break;
            case S_IFREG:
                printf("%s is a regular file.\n", p[i]);
                break;
        }
    }
    return 0;
}
```

### 一个比喻
硬链接类似于C++中的引用(reference)，而软连接则类似于指针(pointer)。
也正是因为此，软连接可以跨文件系统而存在，而硬链接则不可用，且硬链接
不可用链接到目录上，原因明确了吧！

(over)

[link]: http://linuxgazette.net/105/pitcher.html