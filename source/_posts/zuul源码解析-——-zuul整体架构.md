---
title: zuul源码解析 —— zuul整体架构
urlname: source/zuul/architecture
date: 2018-11-29 10:55:13
categories:
    - source
    - zuul
tags:
---


## 工作原理

关于zuul架构，官方的[How-it-Works](https://github.com/Netflix/zuul/wiki/How-it-Works)已经讲得很清楚了，这里简单地翻译一下：

Zuul的核心就是一系列的过滤器，能够在HTTP请求和响应的过程中进行一系列的操作。Zuul提供了一个可以在运行时动态地读取、编译并运行这些过滤器的框架。
以下是过滤器的一些重要属性：  
- **类型**：通常定义了过滤器会作用在路由的哪个阶段
- **执行顺序**：当同一阶段存在多个过滤器时，决定了这些过滤器的执行顺序
- **执行条件**：决定过滤器是否被执行
- **行为**：当符合条件的时候执行的操作

过滤器之间不会直接进行通信，而是通过每个请求唯一对应的RequestContext实例进行状态共享。  
过滤器目前只能通过Groovy语言编写（指的是动态过滤器），尽管从理论上来说Zuul支持所有以JVM为执行环境的语言。  
过滤器的源码应该写到指定的目录中，Zuul Server会周期性地检查这些目录的变化。一旦某些过滤器有更新，它们将会被Zuul读取并动态地编译到运行时环境中，在随后到来的每个请求中都将被Zuul调用。

![zuul_architecture](/images/zuul/architecture.png)

> 图片来源 - [netflix techblog](https://medium.com/netflix-techblog/announcing-zuul-edge-service-in-the-cloud-ab3af5be08ee)

## 核心类

### ZuulServlet

```xml
<!-- web.xml -->
<servlet>
    <servlet-name>ZuulServlet</servlet-name>
    <servlet-class>com.netflix.zuul.http.ZuulServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>ZuulServlet</servlet-name>
    <url-pattern>/*</url-pattern>
</servlet-mapping>
```

Zuul是一个web应用，在web.xml中其使用的Servlet实现是ZuulServlet。因此毫无疑问的，ZuulServlet就是整个Zuul的核心。

来看看servier方法

```java
// ZuulServlet.java

@Override
public void service(javax.servlet.ServletRequest servletRequest, javax.servlet.ServletResponse servletResponse) throws ServletException, IOException {
    try {
        // 初始化当前的zuul request context，将request和response放入上下文中
        init((HttpServletRequest) servletRequest, (HttpServletResponse) servletResponse);

        // Marks this request as having passed through the "Zuul engine", as opposed to servlets
        // explicitly bound in web.xml, for which requests will not have the same data attached
        RequestContext context = RequestContext.getCurrentContext();
        // 为此次请求设置标识
        context.setZuulEngineRan();

        // zuul对请求的处理流程 start
        // 以下几个try块部分是zuul对一个请求的处理流程：pre -> route -> post
        // 可以看到：
        // 1. post是必然执行的（可以类比finally块），但如果在post中抛出了异常，交由error处理完后就结束，避免无限循环
        // 2. 任何阶段抛出了ZuulException，都会交由error处理
        // 3. 非ZuulException会被封装后交给error处理
        try {
            preRoute();
        } catch (ZuulException e) {
            error(e);
            postRoute();
            return;
        }
        try {
            route();
        } catch (ZuulException e) {
            error(e);
            postRoute();
            return;
        }
        try {
            postRoute();
        } catch (ZuulException e) {
            error(e);
            return;
        }
        // zuul对请求的处理流程 end

    } catch (Throwable e) {
        error(new ZuulException(e, 500, "UNHANDLED_EXCEPTION_" + e.getClass().getName()));
    } finally {
        // 此次请求完成，移除相应的上下文对象
        RequestContext.getCurrentContext().unset();
    }
}
```

上面代码逻辑正好zuul官方的架构图相对应

![zuul_architecture1](https://camo.githubusercontent.com/4eb7754152028cdebd5c09d1c6f5acc7683f0094/687474703a2f2f6e6574666c69782e6769746875622e696f2f7a75756c2f696d616765732f7a75756c2d726571756573742d6c6966656379636c652e706e67)


### RequestContext
RequestContext是各filter之间进行消息传递的介质，其本质上是一个Map，这样就可以在上下文中存储任何kv pair。

**TODO::** 这里有一个疑问，为什么RequestContext是继承自ConcurrentHashMap？因为理论上来说RequestContext对象是绝对线程安全的（线程隔离）。

RequestContext包含一个类变量threadLocal，

```java
protected static final ThreadLocal<? extends RequestContext> threadLocal = new ThreadLocal<RequestContext>() {
    @Override
    protected RequestContext initialValue() {
        try {
            // 上下文实例类型取决于contextClass，例如在NFRequestContext中就改写了contextClass
            return contextClass.newInstance();
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

在代码的任何地方都可以通过静态方法getCurrentContext获取到属于当前线程（实际上经过zuul的处理，context实例是按请求隔离的）的RequestContext实例

```java
public static RequestContext getCurrentContext() {
    if (testContext != null) return testContext;

    RequestContext context = threadLocal.get();
    return context;
}
```

#### NFRequestContext

`NFRequestContext`继承自RequestContext，定义了一些与netflix其它组件有关的特定概念和数据的key，例如eureka VIP，Ribbon client返回的repsonse等。它有一个静态方法

```java
// NFRequestContext.java
static {
    RequestContext.setContextClass(NFRequestContext.class);
}
```

也就是说，只要加载了类NFRequestContext，应用中所有RequestContext的实例都将是NFRequestContext类型。

### ZuulRunner

ZuulServlet并不做任何实际的操作，而是将所有操作交给ZuulRunner完成。

而事实上ZuulRunner的大部分操作也是委托给FilterProcessor去完成的，除了init方法。

```java
// ZuulRunner.java
/**
 * sets HttpServlet request and HttpResponse
 */
public void init(HttpServletRequest servletRequest, HttpServletResponse servletResponse) {
    RequestContext ctx = RequestContext.getCurrentContext();
    if (bufferRequests) {
        // wrap request并缓存其body
        // 所谓buffer是指将其内容缓存起来，使得可以安全地重复调用getReader(), getInputStream()等方法
        // 因为一般来说，流操作一次之后就不能重复操作了
        // 类com.netflix.zuul.http.HttpServletRequestWrapper注释上有详解
        ctx.setRequest(new HttpServletRequestWrapper(servletRequest));
    } else {
        ctx.setRequest(servletRequest);
    }

    // response没得选，肯定是wrap过的
    ctx.setResponse(new HttpServletResponseWrapper(servletResponse));
}
```

### FilterProcessor

FilterProcessor是个单例，通过getInstance()方法获取。

FilterProcessor的核心方法有两个：runFilters和processZuulFilter

```java
// FilterProcessor.java
/**
 * 这个是真正执行filter的方法，每调用一次都会执行同一类型的所有filter
 * runs all filters of the filterType sType/ Use this method within filters to run custom filters by type
 *
 * @param sType the filterType.
 * @throws Throwable throws up an arbitrary exception
 */
public Object runFilters(String sType) throws Throwable {
    // 添加debug信息
    if (RequestContext.getCurrentContext().debugRouting()) {
        Debug.addRoutingDebug("Invoking {" + sType + "} type filters");
    }
    boolean bResult = false;
    // 通过FilterLoader获取指定类型的所有filter
    List<ZuulFilter> list = FilterLoader.getInstance().getFiltersByType(sType);
    if (list != null) {
        // 这里没有进行try...catch... 意味着只要任何一个filter执行失败了整个过程就会中断掉
        for (int i = 0; i < list.size(); i++) {
            ZuulFilter zuulFilter = list.get(i);
            Object result = processZuulFilter(zuulFilter);
            if (result != null && result instanceof Boolean) {
                // 注意这里写的是|=不是!=
                // TODO:: 为什么要用这种写法？既然只有result为boolean类型时才执行，直接赋值不行吗
                bResult |= ((Boolean) result);
            }
        }
    }
    return bResult;
}
```

```java
// FilterProcessor.java
/**
 * 执行单个zuul filter
 * Processes an individual ZuulFilter. This method adds Debug information. Any uncaught Thowables are caught by this method and converted to a ZuulException with a 500 status code.
 *
 * @param filter
 * @return the return value for that filter
 * @throws ZuulException
 */
public Object processZuulFilter(ZuulFilter filter) throws ZuulException {

    RequestContext ctx = RequestContext.getCurrentContext();
    boolean bDebug = ctx.debugRouting();
    final String metricPrefix = "zuul.filter-";     // 这个变量没有用到...
    long execTime = 0;
    String filterName = "";
    try {
        long ltime = System.currentTimeMillis();
        filterName = filter.getClass().getSimpleName();

        RequestContext copy = null;
        Object o = null;
        Throwable t = null;

        if (bDebug) {
            Debug.addRoutingDebug("Filter " + filter.filterType() + " " + filter.filterOrder() + " " + filterName);
            // copy了一份context用于debug
            copy = ctx.copy();
        }

        ZuulFilterResult result = filter.runFilter();
        ExecutionStatus s = result.getStatus();

        // 统计执行时间
        execTime = System.currentTimeMillis() - ltime;

        // 这段对过滤器的执行状态进行记录
        switch (s) {
            case FAILED:
                t = result.getException();
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
                break;
            case SUCCESS:
                o = result.getResult();
                ctx.addFilterExecutionSummary(filterName, ExecutionStatus.SUCCESS.name(), execTime);
                if (bDebug) {
                    Debug.addRoutingDebug("Filter {" + filterName + " TYPE:" + filter.filterType() + " ORDER:" + filter.filterOrder() + "} Execution time = " + execTime + "ms");
                    Debug.compareContextState(filterName, copy);
                }
                break;
            default:
                break;
        }

        if (t != null) throw t;

        // 触发一个filter usage回调
        // 当前notifier的实现固定是BasicFilterUsageNotifier，通过Servo统计filter的调用
        usageNotifier.notify(filter, s);
        return o;

    } catch (Throwable e) {
        if (bDebug) {
            Debug.addRoutingDebug("Running Filter failed " + filterName + " type:" + filter.filterType() + " order:" + filter.filterOrder() + " " + e.getMessage());
        }
        usageNotifier.notify(filter, ExecutionStatus.FAILED);
        if (e instanceof ZuulException) {
            throw (ZuulException) e;
        } else {
            ZuulException ex = new ZuulException(e, "Filter threw Exception", 500, filter.filterType() + ":" + filterName);
            // 如果在line35之前抛出了异常，这个execTime的值会是0
            // 不过ZuulFilter.runFilter()中做了try...catch...处理，理论上来说不会出现异常
            ctx.addFilterExecutionSummary(filterName, ExecutionStatus.FAILED.name(), execTime);
            throw ex;
        }
    }
}
```

```java
// FilterProcessor.java
/**
 * Publishes a counter metric for each filter on each use.
 */
public static class BasicFilterUsageNotifier implements FilterUsageNotifier {
    private static final String METRIC_PREFIX = "zuul.filter-";

    @Override
    public void notify(ZuulFilter filter, ExecutionStatus status) {
        // 通过Netflix Servo对每个filter进行调用计数
        DynamicCounter.increment(METRIC_PREFIX + filter.getClass().getSimpleName(), "status", status.name(), "filtertype", filter.filterType());
    }
}
```

### StartServer

StartServer是一个ServletContextListener，负责在web应用启动后执行一些初始化操作

```java
// zuul-netflix-webapp
// StartServer.java
protected void initialize() throws Exception {
    // 这个操作是触发静态变量AmazonInfoHolder.INFO的初始化，并不是没有意义的
    AmazonInfoHolder.getInfo();
    // 监控、度量等初始化
    initPlugins();
    // 动态Filter等相关类的初始化
    initZuul();
    // cassandra初始化
    initCassandra();
    // NIWS: Netflix Internal Web Service
    // 主要是初始化ribbon的客户端之类的
    initNIWS();

    // 初始化完成，修改当前应用实例的状态为up
    ApplicationInfoManager.getInstance().setInstanceStatus(InstanceInfo.InstanceStatus.UP);
}
```


#### 相关链接

- [FilterLoader工作方式](filter/dynamic_load.html#FilterLoader)


