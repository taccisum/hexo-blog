---
title: zuul源码解析目录
urlname: source/zuul/home
date: 2018-11-28 21:25:22
categories:
    - source
    - zuul
tags:
---

## 前言

最近看了[芋道](http://www.iocoder.cn/)的[eureka源码解析](http://www.iocoder.cn/categories/Eureka/)。上面还提到了一些阅读源码的方法，觉得挺有意思的，就想自己尝试一下。正好zuul相对完整系统的源码解析文章在网上还没有，而且zuul的源码相对来说还是比较简单的，所以第一篇源码解析文章就选择了zuul。

此系列文章基于`zuul 1.3.0`。

此系列文章与spring cloud zuul无关，是对原生netflix zuul的源码解析。对于spring cloud zuul的源码解析，网上相关的文章还是比较多的。

在Zuul中，过滤器分为容器（jetty）的过滤器（Filter）和Zuul的过滤器（ZuulFilter）。由于在我们关注的更多的是Zuul的过滤器，因此在文章中如无特殊说明，提到过滤器均指Zuul的过滤器。

## 文章目录

- [zuul源码解析 —— 搭建调试环境](debug_env.html)
- [zuul源码解析 —— zuul整体架构](architecture.html)
- [zuul源码解析 —— 过滤器](filters.html)
- [zuul源码解析 —— Pre和其它类型的过滤器](filters/pre.html)
- [zuul源码解析 —— Route类型过滤器](filters/route.html)
- [zuul源码解析 —— Post类型过滤器](filters/post.html)
- [zuul源码解析 —— 动态加载Filter（未完成）](filter/dynamic_load.html)

## TODO LIST

个人记录的todo list，请无视

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
- [x] zuul的动态配置（DynamicPropertyFactory） - 属于zuul archaius的内容
- [ ] zuul event
- [x] vip, routevip, altvip都是些啥
- [x] RequestContext与NFRequextContext到底用哪个，是在哪里决定的？
- [ ] SurgicalDebugFilter
- [ ] WeightedLoadBalancer
