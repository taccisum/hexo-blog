---
title: zuul源码解析 —— Pre和其它类型的过滤器
urlname: source/zuul/filters/pre
date: 2018-12-01 14:17:36
categories:
    - source
    - zuul
tags:
---


## Pre类型

Pre类型的过滤器执行顺序为：DebugFilter(1) -> Routing(1) -> PreDecoration(20) -> WeightedLoadBalancer(30) -> DebugRequest(10000)

### DebugFilter

判断是否为此次请求开启debug的一个过滤器。

```java
// Debug.groovy
boolean shouldFilter() {
    // 配置中是否开启debug
    if ("true".equals(RequestContext.currentContext.getRequest().getParameter(debugParameter.get())))
        return true;
    return routingDebug.get();
}

Object run() {
    // 设置当前请求的上下文的debug标识为true，作为之后执行filter时是否记录debug信息的依据
    RequestContext.getCurrentContext().setDebugRequest(true)
    RequestContext.getCurrentContext().setDebugRouting(true)
    return null;
}
```

### Routing

Routing过滤器判断是将请求路由到静态资源还是其它服务。

```java
// Routing.groovy
boolean shouldFilter() {
    return true
}

Object staticRouting() {
    // 路由到静态资源
    FilterProcessor.instance.runFilters("healthcheck")
    FilterProcessor.instance.runFilters("static")
}

Object run() {
    staticRouting() //runs the static Zuul

    // TODO:: 这里routeVIP的值是固定的（origin），后面也没有找到修改该值的地方，导致zuul只能路由到某个单一的服务，暂时不知道原因，先mark一下
    // 目标Eureka VIP
    ((NFRequestContext) RequestContext.currentContext).routeVIP = defaultClient.get()
    String host = defaultHost.get()
    if (((NFRequestContext) RequestContext.currentContext).routeVIP == null) ((NFRequestContext) RequestContext.currentContext).routeVIP = ZuulApplicationInfo.applicationName
    if (host != null) {
        final URL targetUrl = new URL(host)
        RequestContext.currentContext.setRouteHost(targetUrl);
        ((NFRequestContext) RequestContext.currentContext).routeVIP = null
    }

    // host与routeVIP不能同时为null
    if (host == null && RequestContext.currentContext.routeVIP == null) {
        throw new ZuulException("default VIP or host not defined. Define: zuul.niws.defaultClient or zuul.default.host", 501, "zuul.niws.defaultClient or zuul.default.host not defined")
    }

    String uri = RequestContext.currentContext.request.getRequestURI()
    // 如果在之前的filter当中给上下文的requestURI赋值了，则覆盖原uri的值
    if (RequestContext.currentContext.requestURI != null) {
        uri = RequestContext.currentContext.requestURI
    }
    if (uri == null) uri = "/"
    if (uri.startsWith("/")) {
        uri = uri - "/"
    }

    // 截取路径的第一段为route
    // TODO:: 这个route有什么用，也暂时没发现
    ((NFRequestContext) RequestContext.currentContext).route = uri.substring(0, uri.indexOf("/") + 1)
}
```


### PreDecoration

预处理过滤器，负责对请求做一些预处理操作，如添加请求头。

```java
// PreDecoration.groovy
@Override
boolean shouldFilter() {
    return true
}

@Override
Object run() {
    if (RequestContext.currentContext.getRequest().getParameter("url") != null) {
        try {
            // routeHost通过请求的参数url指定
            // 如果routeHost有值，则在route阶段会由ZuulHostRequest进行处理
            RequestContext.getCurrentContext().routeHost = new URL(RequestContext.currentContext.getRequest().getParameter("url"))
            // 开启GZip
            RequestContext.currentContext.setResponseGZipped(true)
        } catch (MalformedURLException e) {
            // url格式错误，返回400
            throw new ZuulException(e, "Malformed URL", 400, "MALFORMED_URL")
        }
    }
    setOriginRequestHeaders()
    return null
}

void setOriginRequestHeaders() {
    RequestContext context = RequestContext.currentContext
    context.addZuulRequestHeader("X-Netflix.request.toplevel.uuid", UUID.randomUUID().toString())
    // 添加被代理者的ip地址到XFF
    context.addZuulRequestHeader(X_FORWARDED_FOR, context.getRequest().remoteAddr)
    // 设置Host头
    context.addZuulRequestHeader(X_NETFLIX_CLIENT_HOST, context.getRequest().getHeader(HOST))
    if (context.getRequest().getHeader(X_FORWARDED_PROTO) != null) {
        context.addZuulRequestHeader(X_NETFLIX_CLIENT_PROTO, context.getRequest().getHeader(X_FORWARDED_PROTO))
    }
}
```

### WeightedLoadBalance

TODO:: 加权负载均衡器，似乎与金丝雀发布（灰度发布）有关，暂不深入了解。

### DebugRequest

负责添加Request的debug信息的过滤器

```java
// DebugRequest.groovy
@Override
boolean shouldFilter() {
    return Debug.debugRequest()
}

@Override
Object run() {
    // 获取原始请求
    HttpServletRequest req = RequestContext.currentContext.request as HttpServletRequest

    // 收集客户端ip信息
    Debug.addRequestDebug("REQUEST:: " + req.getScheme() + " " + req.getRemoteAddr() + ":" + req.getRemotePort())
    // 收集HTTP请求行信息
    Debug.addRequestDebug("REQUEST:: > " + req.getMethod() + " " + req.getRequestURI() + " " + req.getProtocol())

    // 收集原请求的请求头信息
    Iterator headerIt = req.getHeaderNames().iterator()
    while (headerIt.hasNext()) {
        String name = (String) headerIt.next()
        String value = req.getHeader(name)
        Debug.addRequestDebug("REQUEST:: > " + name + ":" + value)
    }

    // 收集原请求的请求体信息
    final RequestContext ctx = RequestContext.getCurrentContext()
    if (!ctx.isChunkedRequestBody()) {
        InputStream inp = ctx.request.getInputStream()
        String body = null
        if (inp != null) {
            body = inp.getText()
            Debug.addRequestDebug("REQUEST:: > " + body)

        }
    }
    return null;
}
```

打印出来的调试信息类似下面这样：

```text
REQUEST_DEBUG::REQUEST:: http 0:0:0:0:0:0:0:1:54075
REQUEST_DEBUG::REQUEST:: > GET /auth-center/foo/bar HTTP/1.1
REQUEST_DEBUG::REQUEST:: > Host:localhost:8080
REQUEST_DEBUG::REQUEST:: > Connection:keep-alive
REQUEST_DEBUG::REQUEST:: > Cache-Control:max-age=0
REQUEST_DEBUG::REQUEST:: > Upgrade-Insecure-Requests:1
REQUEST_DEBUG::REQUEST:: > User-Agent:Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/67.0.3396.87 Safari/537.36
REQUEST_DEBUG::REQUEST:: > Accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
REQUEST_DEBUG::REQUEST:: > Accept-Encoding:gzip, deflate, br
REQUEST_DEBUG::REQUEST:: > Accept-Language:zh-CN,zh;q=0.9,zh-TW;q=0.8
REQUEST_DEBUG::REQUEST:: > Cookie:Idea-9d03e8f=36659dd1-27e5-4e9b-8824-19bf2de30b9e; _ga=GA1.1.79701639.1521035841; wcsid=u5MHraIxIjs3Na0G3m39N0Hab53odbCD; hblid=5HLUbXnG2OuboFyp3m39N0HbjAa3Cd5a; _oklv=1542679881769%2Cu5MHraIxIjs3Na0G3m39N0Hab53odbCD; _okdetect=%7B%22token%22%3A%2215426798825920%22%2C%22proto%22%3A%22http%3A%22%2C%22host%22%3A%22localhost%3A4040%22%7D; olfsk=olfsk020058268955480463; _okbk=cd4%3Dtrue%2Cvi5%3D0%2Cvi4%3D1542679883339%2Cvi3%3Dactive%2Cvi2%3Dfalse%2Cvi1%3Dfalse%2Ccd8%3Dchat%2Ccd6%3D0%2Ccd5%3Daway%2Ccd3%3Dfalse%2Ccd2%3D0%2Ccd1%3D0%2C; _ok=1700-237-10-3483; _gauges_unique_month=1; _gauges_unique_year=1; _gauges_unique=1; freeform=3213
REQUEST_DEBUG::REQUEST:: > 
```

## 其它类型

### Options

好像是一个没用的filter...感觉是还未完成？

```java
// Options.groovy
boolean shouldFilter() {
    String method = RequestContext.currentContext.getRequest() getMethod();
    // 处理OPTIONS方法的HTTP请求
    if (method.equalsIgnoreCase("options")) return true;
}

@Override
String uri() {
    return "any path here"
}

@Override
String responseBody() {
    // 啥也不返回
    return "" // empty response
}
```

### Healthcheck

Healthcheck过滤器提供了一个静态资源/healthcheck，方便检测zuul应用是否正常运行。但是功能好像过于弱鸡了... - -

```java
// Healthcheck.groovy
@Override
String filterType() {
    return "healthcheck"
}

@Override
String uri() {
    return "/healthcheck"
}

@Override
String responseBody() {
    // 只返回了一个ok...简单粗暴...
    RequestContext.getCurrentContext().getResponse().setContentType('application/xml')
    return "<health>ok</health>"
}
```

### ErrorResponse

```java
// ErrorResponse.groovy
boolean shouldFilter() {
    // 根据标识判断错误是否已被处理
    return RequestContext.getCurrentContext().get("ErrorHandled") == null
}

Object run() {
    RequestContext context = RequestContext.currentContext
    Throwable ex = context.getThrowable()
    try {
        LOG.error(ex.getMessage(), ex);
        throw ex
    } catch (ZuulException e) {
        String cause = e.errorCause
        if (cause == null) cause = "UNKNOWN"
        // 添加错误原因到请求头X-Netflix-Error-Cause
        RequestContext.getCurrentContext().getResponse().addHeader("X-Netflix-Error-Cause", "Zuul Error: " + cause)
        // 对该次错误请求进行统计
        if (e.nStatusCode == 404) {
            ErrorStatsManager.manager.putStats("ROUTE_NOT_FOUND", "")
        } else {
            ErrorStatsManager.manager.putStats(RequestContext.getCurrentContext().route, "Zuul_Error_" + cause)
        }

        // 判断是否改写响应状态，则请求传入的参数决定
        if (overrideStatusCode) {
            RequestContext.getCurrentContext().setResponseStatusCode(200);
        } else {
            RequestContext.getCurrentContext().setResponseStatusCode(e.nStatusCode);
        }
        // 设置标识，表示不再返回zuul转发请求得到的响应结果（如果有）
        context.setSendZuulResponse(false)
        // 设置zuul的异常响应body
        context.setResponseBody("${getErrorMessage(e, e.nStatusCode)}")
    } catch (Throwable throwable) {
        // 处理未知异常，与处理ZuulException的逻辑大体相同
        RequestContext.getCurrentContext().getResponse().addHeader("X-Zuul-Error-Cause", "Zuul Error UNKNOWN Cause")
        ErrorStatsManager.manager.putStats(RequestContext.getCurrentContext().route, "Zuul_Error_UNKNOWN_Cause")

        if (overrideStatusCode) {
            RequestContext.getCurrentContext().setResponseStatusCode(200);
        } else {
            RequestContext.getCurrentContext().setResponseStatusCode(500);
        }
        context.setSendZuulResponse(false)
        context.setResponseBody("${getErrorMessage(throwable, 500)}")
    } finally {
        // 设置标识，表示错误已经被处理（防止存在多个error过滤器时重复处理）
        context.set("ErrorHandled") //ErrorResponse was handled
        return null;
    }
}
```



