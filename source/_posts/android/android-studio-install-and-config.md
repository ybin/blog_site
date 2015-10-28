title: Android Studio安装和配置
date: 2015-05-06 11:16:03
categories: android
tags: [android, studio]
---

Android Studio(AS) 1.2的安装与配置。以下只针对1.2版本。

<!-- more -->

## 下载 (Hello GFW!)

不用问，只能通过镜像网站下载，感谢[androiddevtools.cn][devtools]，
可以单独下载AS，也可以下载AS与SDK的bundle，注意SDK需要3G磁盘空间，
里面包含完整的`docs`，在AS里面查看文档会很方便(Ctrl + Q)，SDK里面
只包括当前最新的API 22，以及对应的build tools，如果需要其他platform
或者build tool，需要自己补充。话说这么大的SDK空间都被docs和system-images
两个目录占用了。

## 安装

安装时会自动检测旧版本，如果检测到就会问你是否卸载，注意这里并不是说
新旧两个版本不能同时存在，后面还会让你选择新版本的安装目录（这不是
折磨人嘛），另外AS和SDK两个目录单独指定，这里有个bug，比如

```
# AS 目录
.../android-studio

# SDK目录
.../android-studio-sdk
```

即AS和SDK在同一个目录下，结果不允许这么搞，说无法升级，但是如果你把SDK
目录从`android-studio-sdk`改为`sdk`，安装顺利完成，嚓，安装脚本出问题了？！

好了，一路安装下去就好了。

## 修改gradle user home(optional)

像我这种C盘空间不足的人，实在是不适合把gradle user home设置到C盘，
这个目录默认在个人主目录中(即.gradle)，设置一个环境变量改掉它：
`GRADLE_USER_HOME`环境变量设置为你想要的目录即可。

## 修改android studio user home(optional)

默认AS会在个人主目录创建一个`.AndroidStudio`文件夹(beta版本是`.AndroidStudioBeta`)，
这个文件夹也是消耗空间的大户，可以通过修改AS的`idea.properties`文件改掉它：

```bash
#---------------------------------------------------------------------
# Uncomment this option if you want to customize path to IDE config folder. Make sure you're using forward slashes.
#---------------------------------------------------------------------
# idea.config.path=${user.home}/.AndroidStudio.2/config
idea.config.path=D:/androidstudio_user_home/config

#---------------------------------------------------------------------
# Uncomment this option if you want to customize path to IDE system folder. Make sure you're using forward slashes.
#---------------------------------------------------------------------
# idea.system.path=${user.home}/.AndroidStudio.2/system
idea.system.path=D:/androidstudio_user_home/system
```

`idea.properties`文件位于AS的bin目录下。其他路径都是依赖于这两个路径的，所以修改这两个即可。

启动AS之后导入旧版本的配置信息，然后就可以正常使用了。

(over)

[devtools]: http://www.androiddevtools.cn/ 