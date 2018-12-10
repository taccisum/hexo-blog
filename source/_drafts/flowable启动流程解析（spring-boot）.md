---
title: flowable启动流程解析（spring boot）
urlname: sourece/flowable/startup/spring_boot
date: 2018-12-10 17:56:00
categories:
    - source
    - flowable
tags:
---

# flowable启动流程解析（spring  boot）

flowable的spring boot starter提供了零配置集成flowable的功能，主要包括各个engine的自动配置、数据库表的自动创建及流程/表单定义自动部署，其次还有rest-api，spring boot aucuator集成等。

Flowable engine一共有5种：App Cmmn Dmn Form Process(即bpmn)。

## flowable相关的AutoConfiguration

```factories
# flowable-spring-boot-autoconfigure: spring.factories

org.springframework.boot.env.EnvironmentPostProcessor=\
  org.flowable.spring.boot.environment.FlowableDefaultPropertiesEnvironmentPostProcessor,\
  org.flowable.spring.boot.environment.FlowableLiquibaseEnvironmentPostProcessor

# Flowable auto-configurations

org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.flowable.spring.boot.actuate.info.FlowableInfoAutoConfiguration,\
    org.flowable.spring.boot.EndpointAutoConfiguration,\
    org.flowable.spring.boot.RestApiAutoConfiguration,\
    org.flowable.spring.boot.app.AppEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.app.AppEngineAutoConfiguration,\
    org.flowable.spring.boot.ProcessEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.ProcessEngineAutoConfiguration,\
    org.flowable.spring.boot.FlowableJpaAutoConfiguration,\
    org.flowable.spring.boot.form.FormEngineAutoConfiguration,\
    org.flowable.spring.boot.form.FormEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.content.ContentEngineAutoConfiguration,\
    org.flowable.spring.boot.content.ContentEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.dmn.DmnEngineAutoConfiguration,\
    org.flowable.spring.boot.dmn.DmnEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.idm.IdmEngineAutoConfiguration,\
    org.flowable.spring.boot.idm.IdmEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.cmmn.CmmnEngineAutoConfiguration,\
    org.flowable.spring.boot.cmmn.CmmnEngineServicesAutoConfiguration,\
    org.flowable.spring.boot.ldap.FlowableLdapAutoConfiguration,\
    org.flowable.spring.boot.FlowableSecurityAutoConfiguration
```

看起来比较多，不过大多数逻辑还是比较简单的。

## Engine自动配置

各Engine配置方式大同小异，以ProcessEngine为例，其涉及到的主要AutoConfiguration为

- ProcessEngineAutoConfiguration
- ProcessEngineServicesAutoConfiguration

flowable通过这两个AutoConfiguration创建了ProcessEngine的实例供应用使用。

### ProcessEngineAutoConfiguration

ProcessEngineAutoConfiguration类图

![class diagram](1)

在ProcessEngineAutoConfiguration中配置了一个类型为SpringProcessEngineConfiguration的bean，SpringProcessEngineConfiguration类图如下

![class diagram](2)

可见，SpringProcessEngineConfiguration是一个ProcessEngineConfiguration，而ProcessEngineConfiguration正是用于创建ProcessEngine的核心类。

不过这里只是注册了一个bean，并没有调用其buildProcessEngine()方法来创建ProcessEngine。想知道ProcessEngine在何处被创建，可以参考[ProcessEngineServicesAutoConfiguration](#ProcessEngineFactoryBean)

### ProcessEngineServicesAutoConfiguration

这个AutoConfiguration主要负责配置ProcessEngineFactoryBean及各个Service，如RuntimeService/RepositoryService/TaskService等。

Service的配置比较简单，来看到ProcessEngineFactoryBean

#### ProcessEngineFactoryBean

需要注意ProcessEngineFactoryBean是在内部类ProcessEngineServicesAutoConfiguration#StandaloneEngineConfiguration中注册的。

这是一个FactoryBean，我们知道Spring通过FactoryBean的getObject()方法来创建bean，来看看其代码

```java
// ProcessEngineFactoryBean.java
public class ProcessEngineFactoryBean implements FactoryBean<ProcessEngine>, DisposableBean, ApplicationContextAware {
    protected ProcessEngineConfigurationImpl processEngineConfiguration;
    @Override
    public ProcessEngine getObject() throws Exception {
        // ...
        this.processEngine = processEngineConfiguration.buildProcessEngine();
        return this.processEngine;
    }
}
```

可以看到，ProcessEngineFactoryBean通过调用processEngineConfiguration.buildProcessEngine()创建了ProcessEngine的实例。

对于processEngineConfiguration这个对象的构建，可以参考[ProcessEngineAutoConfiguration](#ProcessEngineAutoConfiguration)

## 自动部署

## 数据库表自动创建

**部分通过手动维护，部分通过liquibase维护**

SchemaManager
Command, CommandExecutor, CommandContext, CommandInterceptor(chain), CommnadContextCloseListener

CommandInterceptor chain: log -> transaction -> command context - > transaction context -> invoke

AbstractSqlScriptBasedDbSchemaManager -> ProcessDbSchemaManager -> AbstractEngineConfiguration.schemaManager

#### sql路径

##### 建表脚本

sql路径规则：
org/flowable/{module}/db/{operator}/flowable.{db_type}.{module}.{component}.sql

简而言之，就是在org/flowable/{module}/db目录下

##### liquibase changelog

ACT_XX_DATABASECHANGELOG表中

数据从哪来？

建表脚本中带有一些change log的insert语句

#### 疑问
- [ ] Agenda?


### 自动部署

SpringProcessEngineConfiguration的deployResources字段会被一个job定时执行deploy（TODO::deploy完后好像没有清空？）。

在ProcessEngineAutoConfiguration中会通过discoverDeploymentResources()找到所有的resource，并set到SpringProcessEngineConfiguration的deployResources字段。

## TODO LIST

- [ ] List<EngineConfigurationConfigurer>
- [ ] 
