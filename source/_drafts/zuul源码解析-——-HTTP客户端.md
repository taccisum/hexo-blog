---
title: zuul源码解析 —— HTTP客户端
urlname: source/zuul/http_client
date: 2018-12-01 14:20:10
categories:
    - source
    - zuul
tags:
---

## 简介

zuul使用`Ribbon`作为HTTP客户端，从Eureka获取实例信息进行负载均衡，同时整合了Hystrix，具备熔断机制。



