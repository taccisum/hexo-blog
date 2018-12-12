---
title: flowable源码阅读随手记
urlname: flowable源码阅读随手记
categories:
  - source
  - flowable
date: 2018-12-12 15:15:30
tags:
---

## CommandExecutor

CommandExecutor是一个chain，最后一个executor一般都是CommandInvoker，也即是说在执行真正的command时会先执行之前的所有executor，例如LogExecutor, TransactionExecutor等。

可以简单地认为CommandExecutor是一种AOP的设计，对每一个命令的执行进行切面处理。

Agenda，flowable的命令队列，每次执行命令之前会先检查Agenda队列中有无operations未执行，如果有，先执行这些operations再执行命令。

Operations, 一个Runnable的LinkedList

**疑问**：

- executor是否单例？并发情况下执行命令会不会有问题？

## 异步执行器？



