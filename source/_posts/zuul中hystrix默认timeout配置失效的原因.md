---
title: zuul中hystrix默认timeout配置失效的原因
urlname: zuul_hystrix_default_timeout_config_invalid_reason_research
date: 2018-08-30 15:29:35
categories:
  - java
  - 踩坑日记
tags:
---


# 所处环境
  - `spring boot`: 1.5.9.RELEASE
  - `spring cloud`: Edgware.RELEASE

# 问题描述
根据netflix hystrix官方的描述，可以通过[`hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds`](https://github.com/Netflix/Hystrix/wiki/Configuration#execution.isolation.thread.timeoutInMilliseconds)这个全局配置为所有服务配置默认超时时间。然而在实践中却发现该配置并没有生效（一直是2000ms），但针对各服务进行单独配置却是可以生效的。

# 问题定位
经google发现，hystrix的timeout机制是通过`HystrixTimer`来处理的。

HystrixTimer是一个单例，开发人员可以在执行`HystrixCommand`前通过调用HystrixTimer.getInstance().addTimerListener()方法来添加一个定时的listener，然后在command on completed的时候移除它。相关的代码已经由netflix实现了，可以查看源码[`AbstractCommand.java`](https://github.com/Netflix/Hystrix/blob/master/hystrix-core/src/main/java/com/netflix/hystrix/AbstractCommand.java#L1137)
```java
TimerListener listener = new TimerListener() {
    @Override
    public int getIntervalTimeInMilliseconds() {
        return originalCommand.properties.executionTimeoutInMilliseconds().get();
    }
};

final Reference<TimerListener> tl = HystrixTimer.getInstance().addTimerListener(listener);

// set externally so execute/queue can see this
originalCommand.timeoutTimer.set(tl);

/**
* If this subscriber receives values it means the parent succeeded/completed
*/
Subscriber<R> parent = new Subscriber<R>() {
    @Override
    public void onCompleted() {
        if (isNotTimedOut()) {
            // stop timer and pass notification through
            tl.clear();
            child.onCompleted();
        }
    }

    @Override
    public void onError(Throwable e) {
        if (isNotTimedOut()) {
            // stop timer and pass notification through
            tl.clear();
            child.onError(e);
        }
    }
};
```

从以上代码可以很容易追溯到hystrix超时时间是从originalCommand.properties(`HystrixCommandProperties`)这个对象中获取的，而在spring cloud提供的`AbstractRibbonCommand`中存在以下代码
```java
final HystrixCommandProperties.Setter setter = HystrixCommandProperties.Setter()
        .withExecutionIsolationStrategy(zuulProperties.getRibbonIsolationStrategy()).withExecutionTimeoutInMilliseconds(
                RibbonClientConfiguration.DEFAULT_CONNECT_TIMEOUT + RibbonClientConfiguration.DEFAULT_READ_TIMEOUT);
```
**导致originalCommand.properties在构建时可以获取到一个executionTimeoutInMilliseconds的实例级默认值，从而覆盖掉了全局的executionTimeoutInMilliseconds配置，但实例级的配置并不受影响，因为hystrix property优先级为：**  
1. Global default from code
2. Dynamic global default property
3. Instance default from code
4. Dynamic instance property

# 解决方案

1. 改写HttpClientRibbonCommand的实现
//todo:: 待补充

# 参考链接

- [Hystrix Configurations](https://github.com/Netflix/Hystrix/wiki/Configuration)
- [聊聊hystrix的timeout处理 - segmentfault](https://segmentfault.com/a/1190000015393836)
- [Hystrix工作原理](https://segmentfault.com/a/1190000012439580)
- [ribbon设置url级别的超时时间](https://www.jianshu.com/p/6e2f11821c77)