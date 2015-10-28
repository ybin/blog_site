title: Android Studio简要指南
date: 2015-03-04 15:26:48
categories: android
tags: [android, gradle]
---

使用自定义framework.jar的app项目（如在框架层增加私有接口）如何使用Android Studio？

<!-- more -->

默认情况下，Android Studio（简称AS）是不允许覆盖sdk中的API的，AS不跟adt bundle不一样，
adt bundle基于eclipse，而eclipse运行修改library的顺序，所以我们可以把自己编译的framework.jar
排在sdk的android.jar之前，这样编译时使用的就是我们增加的接口，否则会因为android.jar中
没有相关接口而导致编译无法通过。

AS的gradle插件(com.android.application)默认把android.jar放在首位，而且不允许修改，
所以即使我们通过`dependencies {}`把自定义的framework.jar引进来，编译的时候也不会使用
该jar文件。

### 使用自定义的framework.jar覆盖android.jar

处理方式就是在编译时，把自定义的framework.jar放到boot class的开头，这样可以保证该
jar文件位于android.jar之前。

```groovy build.gradle of module
gradle.projectsEvaluated {
    tasks.withType(JavaCompile) {
        options.compilerArgs << '-Xbootclasspath/p:your/absolute/path/to/framework.jar'
        options.encoding = "GBK" // "UTF-8" etc
    }
}
```

### 定制lint选项

设置lint选项，让lint在发现错误时不停止，这样即使lint有问题，但是编译可以正常进行。

```groovy build.gradle of module
lintOptions {
    abortOnError false
}
``` 

### 定制sourceSet

```groovy build.gradle of module
sourceSets {
    main {
        manifest.srcFile 'AndroidManifest.xml'
        java.srcDirs = ['src']
        resources.srcDirs = ['src']
        aidl.srcDirs = ['src']
        renderscript.srcDirs = ['src']
        res.srcDirs = ['res']
        assets.srcDirs = ['assets']
    }
    androidTest.setRoot('tests')
}
```

这样设置代码结构之后，把module下的`src/`目录删除，然后把原来app的代码复制过来即可，
这样就可以使用以前的目录结构了。

遗留问题：
经过这样的设置之后，编译没有问题，但是在IntelliJ Idea中查看代码的时候，自定义的代码
无法解析，原因就是自定义的framework.jar只是gradle会用到，而IntelliJ Idea并不知道
这个jar文件的存在，所以会出现符号无法解析。

(over)