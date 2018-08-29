---
title: Gradle学习（二） - 常用插件
date: 2017-09-26 17:56:40
categories:
    - java
    - 构建工具
    - gradle
tags:
---

## build-scan（构建审视）插件

### 介绍
> A build scan is a shareable and centralized record of a build that provides insights into what happened and why. By applying the build scan plugin to your project, you can create a build scan in the Gradle Cloud for free.
> [Creating Build Scans](https://guides.gradle.org/creating-build-scans/)

大概的意思是build scan能为你提供构建过程中发生的what and why信息，在你构建的时候，插件会抓取数据提交到`Gradle Cloud`，同时返回一个包含构建信息的链接。

**工作流程**
![overview](https://docs.gradle.com/build-scan-plugin/images/build-scan-service-overview.svg)

### 配置
配置方式很简单，只需要在build.gradle中加入
```groovy
plugins {
    id 'com.gradle.build-scan' version '1.9' 
}
```

添加了插件后，可以通过buildScan块来配置插件，其中有两个`license`相关的属性是**_必需要配置_**的
```groovy
buildScan {
    licenseAgreementUrl = 'https://gradle.com/terms-of-service'
    licenseAgree = 'yes'
}
```

import changes（如果没有勾选auto-import），可以看到Gradle Project面板上多了个task
![](/images/gradle_build_scan_task.png)

**tip**
- 如果添加新的plugin，应该确保build-scan**_总是在第一个位置_**，否则其之前的插件虽然仍然正常工作，但是无法到抓取相关的构建信息

### 使用
先执行build -> build，再执行这个build scan -> buildScanPublishPrevious，不出意外可以看到terminal中返回了一个链接
```text
9:50:39 PM: Executing external task 'buildScanPublishPrevious'...
:buildScanPublishPrevious

Publishing build scan...
https://gradle.com/s/af72je4qbzhme

BUILD SUCCESSFUL

Total time: 1.283 secs
9:50:41 PM: External task execution finished 'buildScanPublishPrevious'.
```

打开链接后可以看到这样一个界面

![build scan](/images/gradle_build_scan_insight.png)

接下来在任一个单元测试内加入一行让构建失败
```java
        Assert.fail();
```
再执行一次构建，打开链接查看
![failed build scan](/images/gradle_build_scan_insight_failure.png)

可以看到有以下几大优点：
- 信息展示非常全面丰富、直观
- 良好的分类、折叠，让用户自己选择展开感兴趣的内容，非常友好
- 网页形式，非常易于分享（这一点有点类似`AngularJS`的错误信息，不过没考察是谁借鉴谁的，或许两者都不是原创？）。

相比之下，这里不得不提到一直被人吐槽的Maven的构建信息，真心是非常不友好😒

**tip**
- 如果构建时出现"There is no previous build data available to publish."，可能是没有先执行任一task。

## Application插件

### 介绍
Application插件可以让你轻松地在本地开发环境下`执行JVM应用`，同时还可以帮助你将应用打包成一个包含了各类操作系统对应`启动脚本`的tar and/or zip文件。

[The Application Plugin](https://docs.gradle.org/3.5/userguide/application_plugin.html)

### 配置
**build.gradle**
```groovy
apply plugin: 'application'
mainClassName = "cn.tac.test.gradle.Application"    //指定程序入口类
//applicationDefaultJvmArgs = ["-Dgreeting.language=en"]      //应用程序启动时的jvm参数
```

添加了Application插件后，项目会多出以下几个task
- run
- startScripts
- installDist
- distZip
- distTar

具体可以通过tasks任务查看

**tips**
- 按照官方的说法，Application插件已经隐式地包括了`Java`插件和`Distribution`插件，因此如果你原来引入了这两个插件，现在可以去掉了

### 使用
以我的main函数为例（注意要跟`mainClassName`属性指定的类一致）
```java
package cn.tac.test.gradle;

import java.util.Arrays;

public class Application {
    public static void main(String[] args) {
        if (args.length > 0) {
             System.out.println("hello, it's you args: " + Arrays.toString(args));
        } else {
             System.out.println("hello, you do not input any args");
        }
    }
}
```

#### 执行应用
```shell
$ sh gradlew run

> Task :run
hello, you do not input any args
```

如果要`传入参数`，可以配置一下run任务
**build.gradle**
```groovy
run {
    if(project.hasProperty("myArgs")){
      args myArgs
    }
}
```

上面配置的意思是，如果当前项目的project对象包含有myArgs属性，那么在执行main函数时就将这个属性作为参数传递，之后我们可以这样执行
```shell
$ sh gradlew run -PmyArgs="123","abc","qaz"

> Task :run
hello, it s you args: [123,abc,qaz]
```

其中-PmyArgs分为两部分
- -P，命令行option。作用是指定一个属性的值run，不能省去
- myArgs，我们刚刚在run任务中自定义的属性，通过-P指定

#### 打包
执行以下脚本可以进行打包
```shell
$ sh gradlew distTar distZip
```

打包好的内容在`/build/distributions`中，分别多了一个tar文件和一个zip文件，解压后查看目录结构如下
```shell
$ tree .
.
├── bin
│   ├── gradle_cli
│   └── gradle_cli.bat
└── lib
    └── gradle_cli-1.0.jar
```

**tips**
- 当然你也可以通过build任务来打包，build任务会自动将`distTar`和`distZip`任务包括进去

