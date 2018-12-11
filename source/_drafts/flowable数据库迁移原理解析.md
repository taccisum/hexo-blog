---
title: flowable数据库迁移原理解析
urlname: flowable数据库迁移原理解析
date: 2018-12-11 16:11:44
categories:
tags:
---


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
