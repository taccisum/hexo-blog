---
title: Gradle学习（五） - 打包发布
urlname: learn_gradle_5
date: 2017-10-13 11:32:20
categories:
    - java
    - 构建工具
    - gradle
tags:
---

## 发布到Maven仓库

### Maven-Publish插件
Gradle将项目发布到Maven仓库需要借助插件，`Maven-Publish`便是官方推荐的一款插件。
[Maven-Publish](https://docs.gradle.org/current/userguide/publishing_maven.html)

### 发布项目
通过`'maven-publish'`引入插件
**build.gradle**
```groovy
apply plugin: 'maven-publish'
```

#### 发布到本地
**build.gradle**
```groovy
apply plugin: 'java'
apply plugin: 'maven-publish'

group = 'cn.tac.test'
version = '1.0'

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
}
publishing {
    repositories {
        maven {
            url "$buildDir/repo"
        }
    }
}
```

在以上配置中
- `publishing{}`是由插件为项目创建的扩展（PublishingExtension类型），这个扩展提供了两个block：`publications{}`（MavenPublication类型）和`repositories{}`（MavenArtifactRepository 类型），分别用于配置`要发布的内容`和`目标仓库`（均可配置多个）。
- `mavenJava{}`被称为发布组件`Component`，用于配置一项要发布的内容，components.java表示这个组件是通过Java插件添加的`java`组件，可以简单地理解为就是将当前java项目作为发布内容。
- `maven{}`指定了一个要发布的目标仓库，当前指定了当前项目的build目录下的`repo`目录，即发布到本地

执行publishing -> publish，可以看到项目成功发布到了`build/repo`目录中。
![build/repo](/images/gradle_learn/publishing/build_repo.png)

**tips**
- 两个`publishing{}`块的内容也可以合在一起写
- mavenJava只是一个命名，并无特殊含义，你也可以将其更换为别的名称，但不能与其它发布组件名称重复
- mavenJava块中还可以配置要发布的artifact的`group`、`id`及`version`，如果不显式指定，则默认采用当前项目的配置

#### 发布源码
在上一步的基础上
1. 新增一个用于获取源码的task
2. 配置发布组件，使其发布的同时额外发布一个包含源码的`artifact`

**build.gradle**
```groovy
……
//这个task可以获取到源码
task sourceJar(type: Jar) {
    from sourceSets.main.allJava
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            ……
            //配置额外发布的artifact
            artifact sourceJar {
                //这个字符串会作为artifact文件的后缀
                classifier "sources"
            }
        }
    }
}
……
```

执行task
![sources](/images/gradle_learn/publishing/build_repo_with_sources.png)

#### 发布到远程仓库
只需要修改url指向远程仓库即可。同时由于远程仓库大多需要认证，因此通常需要通过`credentials{}`指定用户名和密码

**build.gradle**
```groovy
……
publishing {
    repositories {
        maven {
            url "http://172.10.10.66:8081/nexus/content/repositories/releases/"
            credentials {
                username = "admin"
                password = "admin123"
            }
        }
    }
}
……
```

#### 条件发布
这点相信会点groovy的同学都不会觉得难，用`if/else`就可以完成。例如下面的配置实现根据version后缀来决定是发布到`snapshots`还是`releases`仓库中

**build.gralde**
```groovy
……
publishing {
    repositories {
        def NEXUS_URL = "http://172.10.10.66:8081/nexus/content/repositories/releases/"
        if (project.version.endsWith("SNAPSHOT")){
            NEXUS_URL = "http://172.10.10.66:8081/nexus/content/repositories/snapshots/"
        }
        maven {
            url NEXUS_URL
            credentials {
                username = "admin"
                password = "admin123"
            }
        }
    }
}
……
```


#### 发布多项目
父项目的task执行的同时会执行其所有子项目的同一task（最常见的如build任务），因此多项目发布只需要配置好所有子项目的发布配置即可。

**build.gradle**
```groovy
subprojects {
    apply plugin: 'java'
    apply plugin: 'maven-publish'

    group "cn.tac.test"
    version "1.0-SNAPSHOT"

    //因为父项目不需要发布，所以只需要配置子项目的发布配置即可
    publishing {
        publications {
            mavenJava(MavenPublication) {
                from components.java
            }
        }
    }
    publishing {
        repositories {
            maven {
                url "$buildDir/repo"
            }
        }
    }
}
```

执行task，可以看到每个子项目都发布到了其对应的`build/repo`中。

**tips**
- 如果子模块之间互相有依赖，Gradle发布时会自动解析各模块的先后发布顺序，无需我们自己配置

### 常见问题

#### 发布后pom.xml中项目的依赖scope为runtime的问题
在发布的时候发现，项目中通过complie依赖的第三方库或其它模块，在发布后由Gradle生成的`pom.xml`文件中，依赖的scope默认变成了`runtime`，而非我们期望的`compile`

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ……
  <groupId>cn.tac.test</groupId>
  <artifactId>publishing</artifactId>
  <version>1.0-SNAPSHOT</version>
  <dependencies>
    <dependency>
      <groupId>cn.tac.test</groupId>
      <artifactId>module1</artifactId>
      <version>1.0-SNAPSHOT</version>
      <scope>runtime</scope>
    </dependency>
  </dependencies>
</project>
```

为了达到期望的效果，只需要在发布配置中通过`withXML`对pom进行修改即可
**build.gradle**
```groovy
publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java

            pom.withXml {
                asNode().dependencies.'*'.findAll() {
                    it.scope.text() == 'runtime' && project.configurations.compile.allDependencies.find { dep ->
                        dep.name == it.artifactId.text()
                    }
                }.each() {
                    it.scope*.value = 'compile'
                }
            }
        }
    }
}
```

再次发布

```xml
<?xml version="1.0" encoding="UTF-8"?>
  ……
  <groupId>cn.tac.test</groupId>
  <artifactId>publishing</artifactId>
  <version>1.0-SNAPSHOT</version>
  <dependencies>
    <dependency>
      <groupId>cn.tac.test</groupId>
      <artifactId>module1</artifactId>
      <version>1.0-SNAPSHOT</version>
      <scope>compile</scope>
    </dependency>
  </dependencies>
</project>
```

**tips**
- 参考链接 [How to change artifactory runtime scope to compile scope?](https://stackoverflow.com/questions/31069666/how-to-change-artifactory-runtime-scope-to-compile-scope)
- [更多withXML的信息](https://docs.gradle.org/current/dsl/org.gradle.api.publish.maven.MavenPom.html#org.gradle.api.publish.maven.MavenPom:withXml(org.gradle.api.Action)
