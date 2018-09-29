---
title: redisson学习笔记
urlname: redisson学习笔记
date: 2018-09-21 13:57:48
categories:
    - java
    - redisson
tags:
---

# 认识redisson

来看下官方的介绍：
> Redis based In-Memory Data Grid for Java. State of the Art Redis client.

可以知道，redisson是Java的一款redis客户端。

作为redis客户端，它和大名鼎鼎的`jedis`有什么区别呢？redisson的宗旨是促进使用者对redis的**关注分离**，从而让使用者能够将精力更集中地放在处理业务逻辑上。

换句话说，redisson对redis的操作进行了一些更高级的抽象，使得我们能够轻松地实现一些复杂的功能，如一系列的`Java分布式对象`，常用的`分布式服务`等。而作为抽象的代价，就是丢失了对底层细节的掌控。


# Getting Start

redisson官方就支持与spring-boot集成，因此根据[官方文档](https://github.com/redisson/redisson/tree/master/redisson-spring-boot-starter)直接依赖
```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.8.0</version>
</dependency>
```

再配置相关yaml，以最简单的本地单机模式启动

**application.yaml**
```yaml
spring:
  redis:
    redisson:
      config: classpath:redisson.yaml
```

**redisson.yaml**
```yaml
singleServerConfig:
  address: redis://127.0.0.1:6379
```

然而添加依赖后直接启动spring boot应用居然报错了，注入`RedissonClient`失败！

妈耶，Google了一下无果，直接看源码，居然发下`redisson-spring-boot-starter`这个包没有`spring.factories`文件，也即是说`RedissonAutoConfiguration`不会自动加载。。

于是补上
```java
@SpringBootApplication
@ImportAutoConfiguration(RedissonAutoConfiguration.class)
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

虽然不知道redisson官方出于什么原因没有提供`spring.factories`文件，总之再次启动，正常。

# redisson锁

## 实践

### 接口调用限制

# redisson执行lua脚本

redisson提供了很方便地执行lua脚本的方式
```java
redissonClient.getScript().eval(
    RScript.Mode.READ_ONLY, //执行模式
    "return redis.call('get', KEYS[1])",    // 要执行的lua脚本
    RScript.ReturnType.INTEGER, // 返回值类型
    Lists.newArrayList("tac"),  // 传入KEYS
    true, 1L, "hello"   //传入ARGV
)
```

> 更具体可以参考官方文档[脚本执行](https://github.com/redisson/redisson/wiki/10.-%E9%A2%9D%E5%A4%96%E5%8A%9F%E8%83%BD#104-%E8%84%9A%E6%9C%AC%E6%89%A7%E8%A1%8C)

## 踩过的一些坑

### 传入的boolean值参数会变成字符串
假设通过redisson的eval()传入的`ARGV = false`，那么在lua脚本中
```lua
print(type(ARGV[1]))    --输出'string'
```

### 传入数值也会变成字符串
假设通过redisson的eval()传入的`ARGV = 1L`，那么在lua脚本中获取会变成
```lua
print(ARGV[3])   --输出'["java.lang.Long",1]'
```

### 直接从redis.call()获取得值是int型，而在lua中进行了数值操作后得到的值却是long型
例如，以下结果是转换为`java.lang.Integer`
```lua
return redis.call('get', 'tac')
```
而以下结果却是转换为`java.lang.Long`
```lua
local tac = redis.call('get', 'tac')
return tac + 1
```

# 一些小细节

- 注意是redisson，不要写成redission
- redisson提供的所有类都是以R开头的，如RLock

