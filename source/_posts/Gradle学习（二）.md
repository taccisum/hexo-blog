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
![图片](https://docs.gradle.com/build-scan-plugin/images/build-scan-service-overview.svg)

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
- 如果构建时出现"There is no previous build data available to publish."，可能是没有先执行build -> build。

