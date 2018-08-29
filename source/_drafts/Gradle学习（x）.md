---
title: Gradle学习（x） - 常用技巧
urlname: learn_gradle_x
date: 2017-09-27 16:49:10
categories:
    - java
    - gradle
tags:
---

## 指定依赖仓库
Gradle支持三种不同类型的仓库（`Maven`、`Ivy`、`Flat`），均通过`repositories{}`来配置，例如要配置一个Maven仓库，可以使用以下配置
```groovy
repositories {
    mavenLocal()        //将指向本地/.m2/repository
    mavenCentral()      //将指向Maven中央仓库
}
```

如果要修改`本地缓存仓库`的地址，需要修改USER_HOME/.m2下的settings.xml，如果没有settings.xml配置文件，Gradle会使用默认的USER_HOME/.m2/repository地址。
具体可以参考[Gradle 使用Maven本地缓存库](http://www.coderli.com/gradle-maven-local-repositories/)

如果要修改`远程仓库`的地址，可以使用`maven{}`来指定，例如要使用aliyun maven镜像，配置如下
```groovy
repositories {
    mavenLocal()
//    mavenCentral()
    maven {
        url 'http://maven.aliyun.com/nexus/content/groups/public/'
    }
}
```

最终Gradle会按照配置仓库的顺序作为依赖解析的优先级，例如上例的优先级是local repo > aliyun repo

## 修改Gradle init的文件内容
[url](https://yrom.net/blog/2015/02/07/change-gradle-maven-repo-url/)

## 工程迁移（sourceSet）

## 依赖管理Dependency Management
[url](https://stackoverflow.com/questions/9547170/in-gradle-how-do-i-declare-common-dependencies-in-a-single-place)
[url1](https://dzone.com/articles/better-dependency-management)
