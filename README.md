原文地址:[Gradle学习系列（三）：Gradle插件](https://juejin.cn/post/6937940389496094751)

## 简介

Gradle本身只是提供了基本的核心功能，其他的特性比如编译Java源码的能力，编译Android工程的能力等等就需要通过插件来实现了。 要想应用插件，需要把插件应用到项目中，应用插件通过
Project.apply() 方法来完成。 在Gradle中一般有两种类型的插件，分别叫做脚本插件和对象插件。

- 脚本插件   
  是额外的构建脚本，它会进一步配置构建，可以把它理解为一个普通的build.gradle。
- 对象插件   
  又叫做二进制插件，是实现了Plugin接口的类，下面分别介绍如何使用它们。

## 一、常见插件类型

### 1.1 脚本插件

比如我们在项目的根目录创建一个utils.gradle

```groovy
def getxmlpackage(boolean x) {
    def file = new File(project.getProjectDir().getPath() + "/src/main/AndroidManifest.xml");
    def paser = new XmlParser().parse(file)
    return paser.@package
}

ext {
    getpackage = this.&getxmlpackage
}
```

这就是一个简单的脚本插件，然后在app moudle引用这个脚本插件

```groovy
apply from: rootProject.getRootDir().getAbsolutePath() + "/utils.gradle"
```

然后就可以调用脚本插件中的方法了

### 1.2 对象插件

对象插件就是实现了org.gradle.api.plugins接口的插件，对象插件可以分为内部插件和第三方插件。

### 1.3 内部插件

Gradle包中有大量的插件，比如我们经常用的java插件和c的插件，我们可以直接引入

```groovy
apply plugin: 'java'
apply plugin: 'cpp'
```

### 1.4 第三方插件

第三方的对象插件通常是jar文件，要想让构建脚本知道第三方插件的存在，需要使用buildscrip来设置。 在buildscrip中来定义插件所在的原始仓库和插件的依赖
，再通过apply方法配置就可以了。Android Gradle插件也属于第三方插件，如果我们想引入Android Gradle插件，可以这么写：

```groovy
buildscript {
    repositories {
        //配置仓库
        google()
        jcenter()
        mavenCentral()
    }
    dependencies {
        //配置插件依赖
        classpath 'com.android.tools.build:gradle:3.5.3'
    }
}
//然后就可以在需要的地方引入android 插件了
apply plugin: 'com.android.application'

```

### 1.5 自定义对象插件

自定义一个插件主要是实现 org.gradle.api.Plugin

## 二、使用maven创建自定义对象插件

### 2.1、新建一个plugin-example模块，删除java目录，新建groovy和resources文件夹，最终的文件目录如下

[模块文件目录](picture/0101_plugin-example模块文件目录.png)

### 2.2、在groovy目录中新建com/afs/example/plugin/MyPlugin.groovy文件

其中MyPlugin.groovy文件的内容如下:

```groovy
package com.afs.example.plugin

import org.gradle.api.Plugin
import org.gradle.api.Project

class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        println("执行自定义插件");
        project.task("haha") {
            doLast {
                println("执行自定义插件 haha task")
            }
        }
    }
}
```

### 2.3、在resources目录中新建META-INF/gradle-plugins/com.example.plugin.properties文件

其中com.example.plugin.properties文件的文件内容如下:

```properties
implementation-class=com.afs.example.plugin.MyPlugin
```

### 2.4、修改plugin-example模块中build.gradle文件的内容

```groovy
apply plugin: 'groovy'  //必须
apply plugin: 'maven'

dependencies {
    implementation gradleApi() //必须
    implementation localGroovy() //必须
    //如果要使用android的API，需要引用这个，实现Transform的时候会用到
    //implementation 'com.android.tools.build:gradle:3.5.0'
}

repositories {
    mavenCentral() //必须
}

//通过maven将插件发布到本地的脚本配置，根据自己的要求来修改
uploadArchives {
    repositories.mavenDeployer {
        // 配置 pom 信息
        pom.version = '1.0.0'
        pom.artifactId = 'exapmplepluginlocal'
        pom.groupId = 'com.example.plugin'
        // 配置仓库地址
        repository(url: uri('../repo'))
    }
}
```

### 2.5、把我们创建的自定义插件上传到本地仓库，项目根目录repo文件夹中

[上传插件到本地仓库](picture/0102_上传插件到本地仓库.png)

### 2.7、在项目根目录的gradle中添加插件依赖

```groovy
buildscript {

    repositories {
        google()
        jcenter()
        mavenCentral()
        //引入本地仓库
        maven {
            url uri('./repo') //指定本地maven的路径，在项目根目录下
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        //引入本地仓库中的自定义插件依赖
        classpath 'com.example.plugin:exapmplepluginlocal:1.0.0'
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        mavenCentral()
        maven {
            url uri('./repo') //指定本地maven的路径，在项目根目录下
        }
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

### 2.8、在app模块中使用我们的自定义插件

```groovy
apply plugin: 'com.example.plugin'
```

运行app模块的gradle，成功看到日志信息如下：

```cmd
> Configure project :app
执行自定义插件
```