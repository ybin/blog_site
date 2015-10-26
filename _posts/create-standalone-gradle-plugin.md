title: Create standalone Gradle plugin by example
date: 2015-02-27 17:04:28
categories: gradle
tags: [gradle, plugin]
---

通过一个简单的实例，学习如何创建Gradle插件。

<!-- more -->

### Gradle plugin

一个插件就是一个实现了`Plugin`接口的Class而已。所以开发一个插件其实
就是创建一个实现了`Plugin`接口的Class。

Gradle插件有三种实现方式：

1. 直接把Class放到`build.gradle`脚本中
2. 在project的根目录下创建一个名为`buildSrc`的工程作为插件的实现，
Gradle在build时会自动关注这个工程
3. 一个独立的project，它会将插件打包成一个.jar文件

### 创建一个独立的Gradle插件

一个插件就是一个实现了`Plugin`接口的Class而已。

一个独立的Gradle插件其实就是一个普通的`.jar`文件，而且Gradle并不关心
这个jar文件是通过什么语言实现的，可以用Java，也可以用Groovy，甚至可以用
Scala，都可以。所以说开发Gradle插件并没有语言限制。

每个插件都有一个id，这才是Gradle关心的，Gradle会根据plugin id以及plugin version
在classpath中寻找`<plugin id>-<plugin version>.jar`这个文件，找到该文件之后，
Gradle怎么知道jar文件中哪个Class实现了`Plugin`接口呢？

答案是：`META-INF`。Gradle会查找该jar文件的`META-INF/gradle-plugins/<plugin id>.properties`
文件，该文件里面记录了实现`Plugin`接口的Class，

``` com.ybin.myplugin.properties
implementation-class=com.example.greetings.GreetingPlugin
```

注意，plugin id一般是`group.name`这种格式，如`com.google.android`，但是这并不是
强制的，完全可以用`abc`来作为plugin id，由此观之，plugin id里面的group，name
与代码里的package是没啥关系的。

既然如此，我们在一个jar里面放置多个`<plugin id>.properties`文件，让它们的内容完全
一样，这样我们的插件就可以拥有多个plugin id了，是否可行呢？当然可以！

一般，我们用Groovy来开发插件，所以可以把`META-INF/gradle-plugins/<plugin id>.properties`
放到`src/main/resources/`目录下，这样生成jar文件时它就会被打包进去了。

剩下的就是正常开发Groovy项目的步骤，不必多说。

```groovy build.gradle
apply plugin: 'groovy'

version = '1.0'

dependencies {
    compile gradleApi()
    compile localGroovy()
}
```

```groovy GreetingPlugin.groovy
package com.example.greetings

import org.gradle.api.Plugin;
import org.gradle.api.Project;

class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.extensions.create("greeting", GreetingPluginExtension)
        project.task('hello') << {
            println "${project.greeting.message} from ${project.greeting.greeter}"
        }
    }
}

class GreetingPluginExtension {
    String message
    String greeter
}
```

最后，`gradle jar`生成jar文件即可。

### 应用插件

```groovy build.gradle of another project
buildscript {
    repositories {
        maven {
            url uri('path/to/your/jar/related/to/project/dir')
        }
    }
    
    dependencies {
        // 用于查找.jar文件
        classpath 'com.ybin:greetings:1.0'
    }
}

// .jar文件中包含多个<plugin id>.properties文件，
// 则该插件就有多个id
//apply plugin: 'greetings'
apply plugin: 'com.ybin.greetings'

greeting {
    message = 'Hi'
    greeter = 'Gradle'
}

```
