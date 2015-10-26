title: Android studio构建App的不同variants和types
date: 2015-03-19 13:58:59
categories: android
tags: [android, build, android studio]
---

一个App可能有免费版和付费版之分，还可能有Phone何Tablet之分，但是不论什么版本，它们之间都有很大的
相同点，只是很少的地方有差异，它们称为build variants，Android Studio能否同时构建这些variants呢？

另外，任何一个版本都可能有debug何release之分，这叫做build type，不同的type能否共存呢？

<!-- more -->

Android Studio可以同时构建多种variants，每种variant有可以有自己的debug和release之分，而且，
最重要的是，这么多不同的版本可以同时安装到一台设备上，debug和release共存，pro和free同在，完美。

### build types

build type只有两种：debug和release。在build.gradle中配置如下，

```groovy
android {
    buildTypes {
        debug {
            // debug版本自动在applicationId后面加个后缀，
            // 从而使得它可以跟正式版本共存于同一设备
            applicationIdSuffix ".debug"
            versionNameSuffix "-debug"
            // 其他配置项目
        }
        release {
            // ...
        }
    }
}
```

### build variants

build variants可以自己随便定义，如

```groovy build.gradle
android {
    defaultConfig {
        applicationId "com.example.app"
        minSdkVersion 14
        targetSdkVersion 19
        versionCode 1
        versionName "1.0"
    }
    productFlavors {
        flavor1 {
            applicationId "com.example.app.flavor1"
            // versionCode, versionName etc.
        }
        flavor2 {
            applicationId "com.example.app.flavor2"
            // versionCode, versionName etc.
        }
    }
}
```

这样配置之后，在`build variants` tool window就会出现四个不同的版本：

- flavor1Debug
- flavor1Release
- flavor2Debug
- flavor2Release

各种variants可以定义自己的applicationId，versionCode等参数，如果不设置的话就使用defaultConfig中的值。

上面说了参数设置，下面说说代码的目录结构，默认不同的variant(flavor)有各自的代码目录，如此时src目录下
就可以是这样的了：`androidTest/`, `main/`, `flavor1/`, `flavor2/`。当然默认各种flavor没有自己的代码，
而是与`main`共享代码，也就是说编译flavor1时从`flavor1/`和`main/`目录下取代码，编译flavor2时从`flavor2/`
和`main/`下取代码，资源文件可以同名，如同时存在`res/values/string.xml`，编译时自动合并，但是Java类
不能一样，会报重复定义的编译错误。所以，如果同一个类有多个不同的版本时，就需要在不同的flavor中各自
定义，而且main中还不能存在同名类，这点儿要注意。

既然如此，我们也可以单独制定各个flavor的source set，只是多个flavor不要出现同一个文件夹的情况，重复
的部分都放到`main`里面去，没有指定的部分使用默认值，如`src/flavor1/res`是flavor1的默认resource目录，

```groovy build.gradle
    sourceSets {
        flavor1 {
            java.srcDirs = ["src\\flavor1\\src"]
        }
        flavor2 {
            java.srcDirs = ['src\\flavor2\\src']
        }
    }
```

[这里][variants]有一个例子。

注意，因为多个apk共存的本质其实就是在编译时把package name修改为applicationId而已，所以涉及到package name
的地方不要硬编码，如自定义View时如果用到自定义属性，那么xmlns不要硬编码package name，而应该使用`res-auto`，
`xmlns:myapp="http://schemas.android.com/apk/res-auto"`，使用content provider的时候涉及到authority
也要灵活处理之。

[variants]:http://www.techotopia.com/index.php/An_Android_Studio_Gradle_Build_Variants_Example