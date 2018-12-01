---
title: zuul源码解析 —— 搭建调试环境
urlname: source/zuul/debug_env
date: 2018-11-29 10:49:29
categories:
    - source
    - zuul
tags:
---



## 下载源码

> [netflix-zuul](https://github.com/Netflix/zuul)

建议fork一份，方便阅读时随手写些注释提交。

## zuul模块介绍

- zuul-core: 确立zuul整体架构及重要类的api
- zuul-netflix: 提供了对netflix其它组件的集成相关实现，如hystrix, ribbon等。此外还有一些工具，如统计工具。主要是为zuul-netflix-webapp提供支持
- zuul-netflix-webapp: Netflix提供的一个可用于生产环境的应用实现，包括Filter的具体实现，监控，动态Filter管理等功能
- zuul-simple-webapp: 一个简单的示例app，功能比较简单，没有太大的研究价值

## 运行zuul-simple-webapp

直接运行以下命令即可

```bash
$ cd zuul-simple-webapp
$ ../gradlew jettyRun
```

然后可以通过 http://localhost:8080 访问[httpbin](http://localhost:8080)。

> 详细可以参考[zuul-simple-webapp](https://github.com/Netflix/zuul/wiki/zuul-simple-webapp)

### 什么是httpbin

> A simple HTTP Request & Response Service

简单来说就是一个方便测试HTTP请求和响应的各种信息,比如cookie, ip, headers和登录验证等的工具。

## 运行zuul-netflix-webapp

由于这是一个可用于生产环境的应用，在其中集成了eureka等，因此需要运行起来还需要做一定的配置

配置zuul.properties，位于zuul-netflix-webapp/src/resources，这里只列出一些必须要覆盖的项
```properties
# zuul.properties
eureka.serviceUrl.default={你的eureka server注册地址}

# 指定filter存放的目录，由于只是调试代码，直接使用zuul提供的filter就好
zuul.filter.pre.path=src/main/groovy/filters/pre
zuul.filter.routing.path=src/main/groovy/filters/route
zuul.filter.post.path=src/main/groovy/filters/post

# origin client对应的eureka VIP
origin.zuul.client.DeploymentContextBasedVipAddresses=foo
origin.zuul.client.Port=8080
```

此外，建议修改代码打开debug开关

```java
// Debug.groovy
boolean shouldFilter() {
    return true;
}
```

然后运行
```bash
$ cd zuul-netflix-webapp
$ ../gradlew jettyRun
```

不出意外的话，此时你的zuul应用应该已经注册到eureka server中了。
访问 http://localhost:8080/healthcheck ，如果出现ok，说明应用已运行成功
访问 http://localhost:8080/path 可以路由到你的foo服务的path路径中


## debug运行(IntelliJ IDEA)

上面的操作可以运行zuul的webapp，但是还没办法进入断点调试，接下来我们尝试配置用IntelliJ IDEA来进行断点调试。

### 添加gradle run/debug配置

在IDEA的run/debug configurations添加一个Gradle配置，参数如下

- **Gradle project**: zuul-simple-webapp或zuul-netflix-webapp
- **Tasks**: jettyRun
- **VM Options**: -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=7777

然后运行，这时应用会进入无限等待remote连接的状态，因此需要再添加一个remote配置。

### 添加remote run/debug配置

在IDEA的run/debug configrations添加一个Remote配置，参数如下

- **Host**: localhost
- **Port**: 7777（即之前Gradle配置中VM Options的address）

然后debug运行，就可以愉快地进行调试了。

