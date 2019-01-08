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

CommandExecutor采用的是拦截过滤器模式。 TODO:: realy?

在flowable中，CommandExecutor是一个链，链上的每个节点都实现了CommandInterceptor。最后一个executor一般都是CommandInvoker，在执行真正的command时会先执行之前的所有executor，例如LogExecutor, TransactionExecutor等。

可以简单地认为CommandExecutor是一种AOP的设计，对每一个命令的执行进行切面处理。

Agenda，flowable的命令队列，每次执行命令之前会先检查Agenda队列中有无operations未执行，如果有，先执行这些operations再执行命令。

Operations, 一个Runnable的LinkedList，可以看到是 **非线程安全的** 。


**疑问**：

- executor是否单例？并发情况下执行命令会不会有问题？

### CommandContext

Q: CommandContext是在哪里初始化的，是单例吗？其字段是单例吗？

A: CommandContext通过`org.flowable.common.engine.impl.context.Context`获得，以栈的形式存在于ThreadLocal中，这样的设计是为了解决Command中调用其它Command的问题。

### sessions的概念

flowable的sessions一共有三类，以类型为key，分别是DbSqlSession, EntityCache, FlowableEngineAgenda


## 异步执行器？



## 目录

### 核心组件

- 命令Command
  - CommandExecutor
  - CommandContext
  - Agenda与Operations
- Sessions

### 具体功能

#### 创建流程实例

#### 查询

各种查询本质上也是一个Command，通过配置Command字段，最终交给执行器执行。

dbsqlsession, executionDataManager, mybatis的sql session（何处初始化？）, cache（mybatis本身有缓存，为什么还要自己弄一个）

statements的xml文件路径：flowable-engine模块下的org/flowable/db/mapping目录

dbsqlsession通过dbsqlsessionfactory.opensession创建，每次调用都创建新的，不过创建后会缓存在command context中。

mybatis的sqlsession通过dbsqlsessionfactory中存储的mybatis的sqlsessionfactory.opensession获取

sessionfactory初始化的地方在AbstractEngineConfiguration.initDbSqlSessionFactory。

Q: mybatis的sqlsessionfactory是在哪里初始化的？  

A:  

mybatis sqlsessionfactory则是在ProcessEngineConfigurator中（在AppEngineConfiguration中调用）或者在AbstractEngineConfiguration.initSqlSessionFactory（SpringAppEngineConfiguration）。

为什么只有app engine获取到了configurator？因为bean ProcessEngineAutoConfiguration#ProcessEngineAppConfiguration.engineConfigurationConfigurer<SpringAppEngineConfiguration\>
注意区分EngineConfigurator和EngineConfigurationConfigurer

AppEngineServicesAutoConfiguration继承自BaseEngineConfigurationWithConfigurers<SpringAppEngineConfiguration\>，所以它注入的List<EngineConfigurationConfigurer<T\>\>中的T实际为SpringAppEngineConfiguration，在各个engine的AutoConfiguration中都定义了EngineConfigurationConfigurer<SpringAppEngineConfiguration\>的bean。调用invokeConfigurers后，各engine的configurator此时也添加到了SpringAppEngineConfiguration的configurator中，因此在SpringAppEngineConfiguration执行init时其configurators才不为空。

最终在process engine configurer中将spring app engine configuration中初始化的mybatis sqlsessionfactory关联给spring process engine configuration。

因此，其实mybatis的sqlsessionfactory是在spring app engine configuration中初始化的。

其它的engine configuration也是类似的逻辑。

Q: ConfigurationConfigurer与Configurer的区别

A: Configurer是将其它engine的Configuration其中一些字段复制，ConfigurationConfigurer是将其它

