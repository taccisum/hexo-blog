---
title: Gradle学习（三） - 多项目构建
urlname: learn_gradle_3
date: 2017-09-30 10:17:25
categories:
    - java
    - gradle
tags:
---

## 前言

从[第一篇](/2017/09/26/Gradle学习（一）/)中我们知道了如何构建一个单项目，但仅仅这样是不够的。实现多项目的构建有利于`模块化`，如此一来我们便能更好地在一个大型项目中分离我们的关注点。

## Getting Started
### 创建根项目
新建一个文件夹作为根项目目录，执行gradle init
```shell
$ mkdir multi_project
$ gradle init
$ ls
build.gradle    gradle          gradlew         gradlew.bat     settings.gradle
```

这次我们要关注的重点有两个文件
- build.gradle
> 配置一些应用于所有子项目的公共配置
- settings.gradle
> 描述各项目之间的关系

### 创建子项目
在根目录下分别创建sub-project1和sub-project2目录，然后分别为其创建build.gradle
```shell
$ mkdir sub-project1 sub-project2
$ touch sub-project1/build.gradle sub-project2/build.gradle
```

然后在根项目的settings.gradle中添加下面内容以关联子项目
```groovy
include 'sub-project1'
include 'sub-project2'
```

**tips**
- 一定要注意子项目只需为其创建build.gradle即可，而非使用gradle init指令初始化，这点很重要
- 子项目的build.gradle只用于配置该项目特有的一些配置项，公共的配置通过根项目的build.gradle配置

### 公共配置
在根项目的build.gradle中加入以下内容
```groovy
allprojects{
    group = 'cn.tac'
    version = '0.1'
    repositories{
        maven {
            url 'http://maven.aliyun.com/nexus/content/groups/public/'
        }
    }
}

subprojects {
    apply plugin: 'java'
    sourceCompatibility = 1.8
    dependencies {
        testCompile 'junit:junit:4.12'
    }
}
```

- `allprojects{}` 为所有项目添加配置项，所以group、version、repositories都放在了这个block下
- `subprojects{}` 仅仅为当前项目的子项目添加配置项（不包括当前项目本身），所以根项目不需要的内容（如依赖）放在了这个block下

### 编写代码
分别为子项目创建src目录
```shell
$ mkdir -p sub-project1/src/test/java/cn/tac/gradle sub-project2/src/test/java/cn/tac/gradle
```

并创建单元测试
```java
package cn.tac.gradle;

import org.junit.Assert;
import org.junit.Test;

public class SubProject1Test {
    @Test
    public void testSimply() {
        System.out.println("hello, i'm sub project1");
    }
}
```

```java
package cn.tac.gradle;

import org.junit.Assert;
import org.junit.Test;

public class SubProject2Test {
    @Test
    public void testSimply() {
        System.out.println("hello, i'm sub project2");
    }
}
```

### 执行构建
完成了上述步骤之后，不出意外此时的目录结构应该如下
```shell
$ tree .
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
├── sub-project1
│   ├── build.gradle
│   └── src
│       └── test
│           └── java
│               └── cn
│                   └── tac
│                       └── gradle
│                           └── SubProject1Test.java
└── sub-project2
    ├── build.gradle
    └── src
        └── test
            └── java
                └── cn
                    └── tac
                        └── gradle
                            └── SubProject2Test.java
```

接下来我们切换到根目录下，执行
```
$ sh gradlew build
```

再查看子项目的目录，发现分别多了一个build目录，可见在为根项目执行构建任务时，其include的所有子项目也分别进行了构建。以下分别是两个子项目的build reports
![sub project 1](/images/gradle_learn/multi_project/subproject1test.png)
![sub project 2](/images/gradle_learn/multi_project/subproject2test.png)

### 依赖其它项目
在`dependencies{}`中添加依赖即可，例如sub-project2要依赖sub-project1，可以在sub-project2的build.gradle中添加
```groovy
dependencies {
  compile project(':sub-project1')
}
```
