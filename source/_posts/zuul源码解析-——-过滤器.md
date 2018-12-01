---
title: zuul源码解析 —— 过滤器
urlname: source/zuul/filters
date: 2018-12-01 14:12:30
categories:
    - source
    - zuul
tags:
---


## 简介

过滤器是zuul最重要的组件，几乎它所有功能都是通过过滤器来实现的。因此，理解各个过滤器的功能是阅读源码必不可少的环节。

## 过滤器的分类

### 按功能分类

如果按照功能分类，过滤器主要有四大类：pre/route/post/error，它们之间的逻辑关系在[ZuulServlet](architecture.html#ZuulServlet)中有描述。
除此之外你也可以自定义特殊类型的过滤器，比如源码中就有一个healthcheck类型。不过自定义类型的过滤器不会被zuul自动识别，需要使用者手动触发调用。

## ZuulFilter

`ZuulFilter`是所有过滤器的基类，其核心方法是runFilter，应用了模板方法模式，来看下代码

```java
// ZuulFilter.java
public ZuulFilterResult runFilter() {
    ZuulFilterResult zr = new ZuulFilterResult();
    // 当前filter是否被禁用
    if (!isFilterDisabled()) {
        // 是否满足filter执行条件
        if (shouldFilter()) {
            Tracer t = TracerFactory.instance().startMicroTracer("ZUUL::" + this.getClass().getSimpleName());
            try {
                // 具体的filter逻辑执行的地方
                Object res = run();
                // wrap一下结果
                zr = new ZuulFilterResult(res, ExecutionStatus.SUCCESS);
            } catch (Throwable e) {
                t.setName("ZUUL::" + this.getClass().getSimpleName() + " failed");
                zr = new ZuulFilterResult(ExecutionStatus.FAILED);
                zr.setException(e);
            } finally {
                t.stopAndLog();
            }
        } else {
            zr = new ZuulFilterResult(ExecutionStatus.SKIPPED);
        }
    }
}
```

上面代码中的关键方法有两个：shouldFilter()和run()。

这两个方法都是抽象的，需要由具体的过滤器去实现。一般来说，我们在阅读过滤器源码时只需要重点关注这两个方法即可。

## zuul-netflix-webapp提供的过滤器

`zuul-netflix-webapp`模块中提供了一些有用的过滤器，在src/main/groovy/filters目录下

|**名称**|**类别**|**Order**|
|:--|:--:|:--:|
|DebugFilter|`pre`|1|
|Routing|`pre`|1|
|PreDecoration|`pre`|20|
|WeightedLoadBalancer|`pre`|30|
|DebugRequest|`pre`|10000|
|ZuulNFRequest|`route`|10|
|ZuulHostRequest|`route`|100|
|Postfilter|`post`|10|
|RequestEventInfoCollectorFilter|`post`|99|
|sendResponse|`post`|1000|
|Stats|`post`|2000|
|ErrorResponse|`error`|1|
|Options|`static`|0|
|Healthcheck|`healthcheck`|0|

由于篇幅关系，详细的代码分析拆分为以下三个章节：
- [Pre和其它类型过滤器](filters/pre.html)
- [Route类型过滤器](filters/route.html)
- [Post类型过滤器](filters/post.html)

## zuul-simple-webapp

`zuul-simple-webapp`提供的过滤器较为简单，实现的功能基本上就是`zuul-netfilx-webapp`功能的一个子集，因此这里就不列出来了。
