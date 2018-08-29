---
title: 在@MybatisTest中使用通用Mapper出现InstantiationException的问题
urlname: instantiationexception_on_mybatistest
date: 2017-11-15 11:42:36
categories: 
  - java
  - 踩坑日记

tags:
---

## 问题描述
在Spring Boot环境中使用[@MybatisTest](http://www.mybatis.org/spring-boot-starter/mybatis-spring-boot-test-autoconfigure/)注解针对Mapper写单元测试的时候，由于同时引入了[通用Mapper](https://github.com/abel533/Mapper)，在单元测试中调用通用Mapper的方法进行CRUD时会出现`InstantiationException`异常
```java
org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.builder.BuilderException: Error invoking SqlProvider method (tk.mybatis.mapper.provider.base.BaseSelectProvider.dynamicSQL).  Cause: java.lang.InstantiationException: tk.mybatis.mapper.provider.base.BaseSelectProvider

	at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:77)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:446)
	at com.sun.proxy.$Proxy89.selectOne(Unknown Source)
	at org.mybatis.spring.SqlSessionTemplate.selectOne(SqlSessionTemplate.java:166)
	at org.apache.ibatis.binding.MapperMethod.execute(MapperMethod.java:82)
	at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:59)
	at com.sun.proxy.$Proxy91.selectByPrimaryKey(Unknown Source)
	at com.ikentop.biz.provider.mapper.hh.ImageRecordMapperTest.testSimply(ImageRecordMapperTest.java:33)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
	at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
	at org.springframework.test.context.junit4.statements.RunBeforeTestMethodCallbacks.evaluate(RunBeforeTestMethodCallbacks.java:75)
	at org.springframework.test.context.junit4.statements.RunAfterTestMethodCallbacks.evaluate(RunAfterTestMethodCallbacks.java:86)
	at org.springframework.test.context.junit4.statements.SpringRepeat.evaluate(SpringRepeat.java:84)
	at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:325)
	at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:252)
	at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:94)
	at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
	at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
	at org.springframework.test.context.junit4.statements.RunBeforeTestClassCallbacks.evaluate(RunBeforeTestClassCallbacks.java:61)
	at org.springframework.test.context.junit4.statements.RunAfterTestClassCallbacks.evaluate(RunAfterTestClassCallbacks.java:70)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
	at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.run(SpringJUnit4ClassRunner.java:191)
	at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
	at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:68)
	at com.intellij.rt.execution.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:51)
	at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:237)
	at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:70)
Caused by: org.apache.ibatis.builder.BuilderException: Error invoking SqlProvider method (tk.mybatis.mapper.provider.base.BaseSelectProvider.dynamicSQL).  Cause: java.lang.InstantiationException: tk.mybatis.mapper.provider.base.BaseSelectProvider
	at org.apache.ibatis.builder.annotation.ProviderSqlSource.createSqlSource(ProviderSqlSource.java:103)
	at org.apache.ibatis.builder.annotation.ProviderSqlSource.getBoundSql(ProviderSqlSource.java:73)
	at org.apache.ibatis.mapping.MappedStatement.getBoundSql(MappedStatement.java:292)
	at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:81)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:148)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:141)
	at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:77)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:433)
	... 34 more
Caused by: java.lang.InstantiationException: tk.mybatis.mapper.provider.base.BaseSelectProvider
	at java.lang.Class.newInstance(Class.java:427)
	at org.apache.ibatis.builder.annotation.ProviderSqlSource.createSqlSource(ProviderSqlSource.java:85)
	... 45 more
Caused by: java.lang.NoSuchMethodException: tk.mybatis.mapper.provider.base.BaseSelectProvider.<init>()
	at java.lang.Class.getConstructor0(Class.java:3082)
	at java.lang.Class.newInstance(Class.java:412)
	... 46 more
```

单元测试代码如下：
```java
@RunWith(SpringRunner.class)
@MybatisTest
public class ImageRecordMapperTest {
    @Autowired
    private ImageRecordMapper mapper;

    @Test
    public void testSimply() {
        ImageRecord record = mapper.selectByPrimaryKey("1");
        Assert.assertNotNull(record);
        Assert.assertEquals("taccisum", record.getBucket());
        Assert.assertEquals("image1", record.getKey());
        Assert.assertEquals("hash1", record.getHash());
    }
}
```

## 分析原因
在github上mapper的[issue](https://github.com/abel533/MyBatis-Spring-Boot/issues/34)中有一段作者的回复，说此异常是由于`通用方法没有被正确初始化`导致。
我这边的情况虽然和这个issue不同，但根本原因是一样的。
查阅`Mybatis-Test`的文档可知，`@MybatisTest`注解不同于`@SpringBootTest`，它只加载保证Mybatis正常运行所需要的configuration。显然通用Mapper作为Mybatis第三方的扩展，并没有被纳入其中，因而默认情况下通用Mapper的starter是不会被加载的，也就导致通用方法不会被初始化。

> The @MybatisTest can be used if you want to test MyBatis components(Mapper interface and SqlSession). By default it will configure MyBatis(MyBatis-Spring) components(SqlSessionFactory and SqlSessionTemplate), configure MyBatis mapper interfaces and configure an in-memory embedded database. MyBatis tests are transactional and rollback at the end of each test by default, for more details refer to the relevant section in the Spring Reference Documentation. Also regular @Component beans will not be loaded into the ApplicationContext.
> 以上是来自[mybatis-spring-boot-test-autoconfigure](http://www.mybatis.org/spring-boot-starter/mybatis-spring-boot-test-autoconfigure/)官方文档的描述

## 解决方案
显然，只要让Mapper的starter在MybatisTest的单元测试启动时得以加载即可解决我们的问题。那么如何做呢？
Spring Boot提供了一个`@ImportAutoConfiguration`的注解，因此只需要在单元测试的目标类上使用该注解将MapperAutoConfiguration导入即可
```java
@RunWith(SpringRunner.class)
@ImportAutoConfiguration(MapperAutoConfiguration.class)
@MybatisTest
public class ImageRecordMapperTest {
	……
}
```

其中`MapperAutoConfiguration`是通用Mapper的starter的auto-configure类。
