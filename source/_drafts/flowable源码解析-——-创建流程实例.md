---
title: flowable源码解析 —— 创建流程实例
urlname: sourece/flowable/start_process_instance
categories:
  - source
  - flowable
date: 2018-12-12 18:30:54
tags:
---

flowable创建完一个流程后，如果启动流程，会一直执行到需要暂停的节点

触发事件：流程创建、实体初始化、流程启动

flowable可以通过RuntimeService的startProcessInstanceByXXX方法来创建及启动流程实例，虽然重载很多，不过逻辑大同小异。这里将以startProcessInstanceByKey()为例进行分析。




