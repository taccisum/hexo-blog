---
title: zuul源码解析
urlname: source/zuul
date: 2018-11-28 21:25:22
categories:
    - source
    - zuul
tags:
---

## 前言

最近看了[芋道](http://www.iocoder.cn/)的[eureka源码解析](http://www.iocoder.cn/categories/Eureka/)，觉得挺有意思的，就想自己尝试一下。正好zuul相对完整系统的源码解析文章在网上还没有，而且zuul的源码相对来说还是比较简单的，所以第一篇源码解析文章就选择了zuul。

TODO::

该系列文章对zuul1的解析，基于zuul版本1.3.0

zuul的代码风格充满着诡异...

此系列文章与spring cloud zuul无关，是对原生netflix zuul的源码解析。
TODO:: 后续可能会出spring cloud zuul的源码解析，但也不一定，因为网上关于这一块的文章还是比较多的

在Zuul中，过滤器分为容器（jetty）的过滤器（Filter）和Zuul的过滤器（ZuulFilter）。由于在我们关注的更多的是Zuul的过滤器，因此在文章中如无特殊说明，提到过滤器均指Zuul的过滤器。

## 目录

- 搭建调试环境
- zuul整体架构
- RequestContext
- 动态加载过滤器 
- Netfilx Filters介绍
- Netfilx Filters - ZuulHostRequest
- Netfilx Filters - Pre类型过滤器 [DebugFilter(1), Routing(1), PreDecoration(20), WeightedLoadBalancer(30), DebugRequest(10000)]
- Netflix Filters - Route类型过滤器
- Netflix Filters - Post类型过滤器
- HTTP客户端
- zuul事件
- Spring Cloud Zuul

## zuul整体架构


## 要点

- [ ] monitor, tracer和counter - 都是netflix servo的东西
- [x] filter registry
- [ ] groovy filter manager: FilterFileManager, FilterLoader
- [x] filter loader
- [x] filter usage notifier
- [x] groovy的filter如何执行？
- [x] zuul的配置与ServletConfig
- [x] zuul servlet, zuul runner, filter processor
- [x] httpbin
- [x] 贯串整个请求的RequestContext
- [x] pre, route, post, error
- [ ] zuul的request wrapper有何不同？
- [x] zuul对一次请求的处理流程
- [x] Debug类
- [ ] RequestContext的key清单
- [x] zuul的动态配置（DynamicPropertyFactory） - 属于zuul archaius的内容
- [ ] zuul event
- [x] vip, routevip, altvip都是些啥
- [x] RequestContext与NFRequextContext到底用哪个，是在哪里决定的？
- [ ] SurgicalDebugFilter这个过滤器有啥用？
- [ ] WeightedLoadBalancer这个过滤器如何使用？

## Servlet
- com.netflix.zuul.http.ZuulServlet
- com.netflix.zuul.scriptManager.FilterScriptManagerServlet
- filterLoader

## 容器filters
- com.netflix.zuul.context.ContextLifecycleFilter
- com.google.inject.servlet.GuiceFilter

## RequestContext keys

- executedFilters - 过滤器执行情况
- debugRouting - 是否debug请求过程
- requestURI - 用来改写请求的uri
- responseGZipped - 为转发的请求添加了`accept-encoding:deflate, gzip`头
- requestEntity - 
- routeHost - 
- sendZuulResponse - 是否发送zuul响应，如果为host，则不进行转发

## zuul debug信息种类

- routingDebug - debugRouting
- requestDebug - debugRequest
