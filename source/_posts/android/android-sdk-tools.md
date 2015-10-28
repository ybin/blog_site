title: Android SDK Tools
date: 2015-03-06 10:00:54
categories: android
tags: [android, sdk]
---

Android SDK中各种tool的用途。

<!-- more -->

### platform

根据不同API的版本，platform有多个版本，即一个API版本对应一个platform，platform的版本有
4.0, 4.2, 4.4, 5.0等，对应的API版本有19, 20, 21等，他们是一一对应的。

platform放在`paltforms/`目录中。

### platform tools

针对不同的platform，SDK提供了一堆的工具，如编译工具，打包工具，签名工具，连接platform(即设备，
任何一个设备都有一个唯一的platform)的工具(如adb, fastboot等)，这些工具放到`platform-tools/`
目录中。

你可能会认为，不同平台提供一套不同的platform tools，但是事实不是这样的，platform tools
针对的是最新平台(platform)的一套工具，这套工具一般是向下兼容的，所以这个目录中保存的应该是
最新的版本。

### build tools

build tools也叫做develop tools，即ADT。

platform tools提供了很多工具，他们是针对最新的平台的最新版本。但是工具是无法向下兼容的，
这部分体现在编译工具上，所以SDK把编译工具单独独立出来，放到`build-tools`目录中，如aapt,
dx等工具。这部分工具因为兼容性，可能存在多个版本，所以build-tools目录下是多个子目录，一个
子目录代表一个版本。

不同的platform或者第三方集成应用可能需要特定的build tools版本，所以就会出现使用某个platform
版本时要求build tools最低不能低于某个版本的情况。不过一般情况下使用最新的版本就好了。

### sdk tools

`tools/`目录下存放的是一些通用的工具，如ant脚本，`android.bat`，9-patch绘制工具等，这部分应该
始终使用最新的版本。

总而言之，`tools/`, `platform-tools/`两个目录应该始终使用最新的版本，而`platforms/`和`build-tools/`
两个目录可以选择性的使用不同的版本。但是这两个版本要匹配才行。

(over)