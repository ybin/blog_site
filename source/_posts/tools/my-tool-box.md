title: 我的工具箱
date: 2014-11-27 10:55:58
categories: 工具
tags: 工具
---

记录自己使用的各种工具，包括但不限于软件工具。

<!--more-->

#### 作图工具
##### 作图工具之： Graphviz

##### 作图工具之： ImageMagick

##### 作图工具之： picpick

##### 作图工具之： PlantUML
[PlantUML][plantuml]是一个用Java实现的开源UML作图工具，基于Graphviz，提供Eclipse插件。

```dot 示例代码: plantuml.txt
@startuml
scale 1.5
title How to lock an object in JamVM\n
start
#YellowGreen:read object's lockword;
if (object locked ?) then (yes)
    if (locked by myself ?) then (yes)
        if (lock too much times ?) then (yes)
            #YellowGreen:lock the monitor;
            #YellowGreen:inflate thin lock;
        else (no)
            #YellowGreen:add lock count;
        endif
    else (no)
        #YellowGreen:lock the monitor;
        while (is it a thin lock ?) is (yes)
            #YellowGreen:set FLC bit;
            if (try to lock) then (success)
                #YellowGreen:inflate thin lock;
            else (fail)
                #YellowGreen:wait on monitor;
            endif
        endwhile (no)
        #YellowGreen:while loop finished;
    endif
else (no)
    #YellowGreen:lock it and return;
endif
stop
@enduml
```

使用命令行编译：`java -jar plantuml.jar plantuml.txt`，也可使用Eclipse插件实时渲染。

UML图(Activity Diagram)显示如下，

![PlantUML activity diagram](/res/img/plantuml.png)

##### 作图工具之： TikZ
[TikZ][tikz]是一个Latex的package，在制作pdf文档或者beamer slides时生成图片非常方便，
而且它足够强大以至于只有你想不到，没有它做不出的图。但是它不是独立的软件，只能依赖
Latex而生。

#### 命令行工具集： GoW(GNU on Windows)
[GoW][gow]是一个命令行集合，正如其名所示，这是GNU tools的Windows移植版，它不需要
Cygwin、不需要MinGW，它只是一个简单的安装包而已，find, grep, awk, sed, ..., 值得拥有。

#### 编程工具
##### 编程工具之： Tiny C Compiler(tcc)
[TCC][tcc]是一个开源的C编译器，兼容ISO C99标准，它的最大特点就是简单。

#### 版本控制
##### 版本控制之： VisualSVN

#### 文档和电子书
##### pandoc

##### SumatraPDF


#### 其他工具
##### EXIF信息读写工具： exiftool
[ExifTool][exiftool]用Perl实现的用于读、写EXIF信息工具，支持但不仅限于EXIF信息，跨平台。


##### KeyTweak



[plantuml]: http://plantuml.sourceforge.net
[exiftool]: http://www.sno.phy.queensu.ca/~phil/exiftool/
[tikz]: http://www.texample.net
[gow]: https://github.com/bmatzelle/gow
[tcc]: http://bellard.org/tcc/
