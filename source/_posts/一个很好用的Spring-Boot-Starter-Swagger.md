---
title: 一个很好用的Spring Boot Starter Swagger
urlname: my_swagger_spring_boot_starter
date: 2019-02-14 14:17:43
categories:
    - framework
    - starter
tags:
---

# Spring Boot Starter Swagger

## 简介

Swagger与Spring Boot现在在Java Web开发领域是再常用不过的两个框架了。集成这两者的Starter现在在Github上也存在很多（基本都是非官方的，官方好像没有提供Starter），但是大多数或多或少都存在以下问题：

- **整合程度低**，许多Springfox-Swagger2提供的功能无法通过Spring Boot的方式进行配置或者配置方式复杂
- **项目疏于维护**，许多Issue没人去解决
- **扩展性不足**，当用户出现一些需求时只能祈求项目更新或者自行修改源码
- **无法同时兼容Spring Boot1.x和Spring Boot2.x**

我曾在Github上找了许久都没找到，索性自行开发了一个。

这个Starter具有以下特点：

- 完美适配springfox-swagger2，几乎支持通过yaml文件进行所有配置
- 提供拦截器，允许用户自行扩展自定义配置
- 同时支持spring boot1和spring boot2
- 扩展了一些小功能，如展示当前hostname等

[**项目地址**](https://github.com/taccisum/spring-boot-starter-swagger)

## 如何使用

引入依赖

**pom.xml**
```xml
<!-- spring boot1用户 -->
<dependency>
    <groupId>com.github.taccisum</groupId>
    <artifactId>swagger-spring-boot1-starter</artifactId>
    <version>{lastest.version}</version>
</dependency>

<!-- spring boot2用户 -->
<dependency>
    <groupId>com.github.taccisum</groupId>
    <artifactId>swagger-spring-boot2-starter</artifactId>
    <version>{lastest.version}</version>
</dependency>
```

**application.yml**
```yaml
swagger:
  base-package: com.github.taccisum.controller
```

启动项目，打开 http://localhost:8080/swagger-ui.html 即可查看API文档。

更多功能可以到 https://github.com/taccisum/spring-boot-starter-swagger 了解。
