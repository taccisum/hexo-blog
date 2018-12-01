---
title: zuul源码解析 —— 动态加载Filter
urlname: source/zuul/filter/dynamic_load
date: 2018-12-01 14:19:14
categories:
    - source
    - zuul
tags:
---


## 介绍

zuul支持用两种语言编写的过滤器，分别是Groovy和Java。不过只有Groovy编写的过滤器才支持动态加载。

所谓动态加载，即是可以在应用的运行时对filter进行CRUD的操作，以达到动态调整zuul行为的目的。

以下我们看看zuul是如何实现这一点的。

需要说明的是，Filter动态加载并不是zuul-core提供的功能，而是zuul-netflix提供的一个实现。在zuul-core中，仅是提供了动态加载filter的可能性。也因此，这里TODO::一下，spring cloud zuul中可能没有这个功能？


## FilterLoader
```java
// FilterLoader.java
    /**
     * 获取所有指定类型的filter并排序，通过ZuulFilter.filterOrder()方法
     * Returns a list of filters by the filterType specified
     */
    public List<ZuulFilter> getFiltersByType(String filterType) {
        // 尝试从缓存中获取，如果存在缓存，则直接返回
        List<ZuulFilter> list = hashFiltersByType.get(filterType);
        if (list != null) return list;

        list = new ArrayList<ZuulFilter>();

        // 从registry获取所有filter，从中找出需要的filter，最终进行排序
        Collection<ZuulFilter> filters = filterRegistry.getAllFilters();
        for (Iterator<ZuulFilter> iterator = filters.iterator(); iterator.hasNext(); ) {
            ZuulFilter filter = iterator.next();
            if (filter.filterType().equals(filterType)) {
                list.add(filter);
            }
        }
        Collections.sort(list); // sort by priority

        // 将结果缓存起来
        hashFiltersByType.putIfAbsent(filterType, list);
        return list;
    }
```

从上面代码可以看到，`getFiltersByType()`依赖于对象filterRegistry，zuul正是通过对该registry的CRUD实现动态加载filter的功能。

filterRegistry是FilterRegistry的一个唯一实例（单例模式）。FilterRegistry的代码很简单，就是简单地维护了一个用于存放filter的ConcurrentHashMap而以。关键需要找出zuul是如何维护这份registry的。

## FilterScriptManagerServlet

需要在运行时维护registry，必然需要有一个入口。这个入口就是`FilterScriptManagerServlet`，这是一个HttpServlet，提供了以下HTTP资源：

|路径|请求方式|Servlet对应的处理方法|描述|
|--|--|--|--|
|LIST|GET|handleListAction|获取filter脚本|
|DOWNLOAD|GET|handleDownloadAction|下载脚本|
|UPLOAD|PUT/POST|handleUploadAction|上传脚本|
|ACTIVATE|PUT/POST|handleActivateAction|启用脚本|
|CANARY|PUT/POST|handleCanaryAction|TODO::不知道干啥的|
|DEACTIVATE|PUT/POST|handledeActivateAction|禁用脚本|






