---
title: zuul源码解析 —— Post类型过滤器
urlname: source/zuul/filters/post
date: 2018-12-01 14:17:54
categories:
    - source
    - zuul
tags:
---

## Post类型过滤器

Post类型过滤器执行顺序为：[Postfilter](#Postfilter)(10) -> [RequestEventInfoCollectorFilter](#RequestEventInfoCollectorFilter)(99) -> [sendResponse](#sendResponse)(1000) -> [Stats](#Stats)(2000)

### Postfilter

后置处理过滤器，主要负责对响应内容进行修饰（如添加响应头）。

```java
// PostDecoration.groovy
boolean shouldFilter() {
    // 判断请求是否是从zuul路由到自身的
    if (true.equals(NFRequestContext.getCurrentContext().zuulToZuul))
        return false; //request was routed to a zuul server, so don't send response headers
    return true
}

Object run() {
    // 为响应添加一些header
    addStandardResponseHeaders(RequestContext.getCurrentContext().getRequest(), RequestContext.getCurrentContext().getResponse())
    return null;
}

void addStandardResponseHeaders(HttpServletRequest req, HttpServletResponse res) {
    println(originatingURL)

    String origin = req.getHeader(ORIGIN)   // 没用到
    RequestContext context = RequestContext.getCurrentContext()
    List<Pair<String, String>> headers = context.getZuulResponseHeaders()
    headers.add(new Pair(X_ZUUL, "zuul"))   // X-Zuul
    headers.add(new Pair(X_ZUUL_INSTANCE, System.getenv("EC2_INSTANCE_ID") ?: "unknown"))   // X-Zuul-instance
    headers.add(new Pair(CONNECTION, KEEP_ALIVE))   // Connection
    headers.add(new Pair(X_ZUUL_FILTER_EXECUTION_STATUS, context.getFilterExecutionSummary().toString()))   // X-Zuul-Filter-Executions
    headers.add(new Pair(X_ORIGINATING_URL, originatingURL))    // X-Originating-URL

    if (context.get("ErrorHandled") == null && context.responseStatusCode >= 400) {
        headers.add(new Pair(X_NETFLIX_ERROR_CAUSE, "Error from Origin"))   // X-Netflix-Error-Cause
        ErrorStatsManager.manager.putStats(RequestContext.getCurrentContext().route, "Error_from_Origin_Server")
    }
}
```

### RequestEventInfoCollectorFilter

这个过滤器负责收集要发送到ESI, EventBus, Turbine等的数据，例如此次请求的数据和当前应用实例的数据。

TODO:: 虽然zuul收集了这些数据，但是并没有找到在哪里使用... mark一下

```java
// RequestEventInfoCollector.groovy
boolean shouldFilter() {
    return true
}

Object run() {
    NFRequestContext ctx = NFRequestContext.getCurrentContext();
    final Map<String, Object> event = ctx.getEventProperties();

    try {
        // 往eventProperties中写入与此次请求有关的数据
        captureRequestData(event, ctx.request);
        // 往eventProperties中写入与当前实例有关的数据
        captureInstanceData(event);
    } catch (Exception e) {
        event.put("exception", e.toString());
        LOG.error(e.getMessage(), e);
    }
}
```

capture data这两个方法比较啰嗦，不过逻辑并不复杂，就是收集各种数据并写入map中

```java
// RequestEventInfoCollector.groovy
void captureRequestData(Map<String, Object> event, HttpServletRequest req) {
    try {
        // 写入请求基本信息
        // basic request properties
        event.put("path", req.getPathInfo());
        event.put("host", req.getHeader("host"));
        event.put("query", req.getQueryString());
        event.put("method", req.getMethod());
        event.put("currentTime", System.currentTimeMillis());

        // 写入请求头
        // request headers
        for (final Enumeration names = req.getHeaderNames(); names.hasMoreElements();) {
            final String name = names.nextElement();
            final StringBuilder valBuilder = new StringBuilder();
            boolean firstValue = true;
            for (final Enumeration vals = req.getHeaders(name); vals.hasMoreElements();) {
                // only prepends separator for non-first header values
                if (firstValue) firstValue = false;
                else {
                    valBuilder.append(VALUE_SEPARATOR);
                }

                valBuilder.append(vals.nextElement());
            }

            event.put("request.header." + name, valBuilder.toString());
        }

        // 写入请求参数
        // request params
        final Map params = req.getParameterMap();
        for (final Object key : params.keySet()) {
            final String keyString = key.toString();
            final Object val = params.get(key);
            String valString;
            if (val instanceof String[]) {
                final String[] valArray = (String[]) val;
                if (valArray.length == 1)
                    valString = valArray[0];
                else
                    valString = Arrays.asList((String[]) val).toString();
            } else {
                valString = val.toString();
            }
            event.put("param." + key, valString);

            // some special params get promoted to top-level fields
            if (keyString.equals("esn")) {
                event.put("esn", valString);
            }
        }

        // 写入响应头
        // response headers
        NFRequestContext.getCurrentContext().getZuulResponseHeaders()?.each { Pair<String, String> it ->
            event.put("response.header." + it.first().toLowerCase(), it.second())
        }
    } finally {
    }
}

private static final void captureInstanceData(Map<String, Object> event) {
    try {
        final String stack = ConfigurationManager.getDeploymentContext().getDeploymentStack();
        if (stack != null) event.put("stack", stack);

        // TODO: add CLUSTER, ASG, etc.

        // 获取此实例（zuul）的信息
        final InstanceInfo instanceInfo = ApplicationInfoManager.getInstance().getInfo();
        // 写入实例信息，id和metadata等
        if (instanceInfo != null) {
            event.put("instance.id", instanceInfo.getId());
            for (final Map.Entry<String, String> e : instanceInfo.getMetadata().entrySet()) {
                event.put("instance." + e.getKey(), e.getValue());
            }
        }

        // AWS相关，跳过
        // caches value after first call.  multiple threads could get here simultaneously, but I think that is fine
        final AmazonInfo amazonInfo = AmazonInfoHolder.getInfo();

        for (final Map.Entry<String, String> e : amazonInfo.getMetadata().entrySet()) {
            event.put("amazon." + e.getKey(), e.getValue());
        }
    } finally {
    }
}
```

### sendResponse

sendResponse是比较重要的一个过滤器，负责将响应写回请求来源。

```java
// sendResponse.groovy
boolean shouldFilter() {
    return !RequestContext.currentContext.getZuulResponseHeaders().isEmpty() ||
            RequestContext.currentContext.getResponseDataStream() != null ||
            RequestContext.currentContext.responseBody != null
}

Object run() {
    // 添加一些响应头并收集响应debug信息到上下文
    addResponseHeaders()
    // 写响应流
    writeResponse()
}

void writeResponse() {
    RequestContext context = RequestContext.currentContext

    // there is no body to send
    if (context.getResponseBody() == null && context.getResponseDataStream() == null) return;

    HttpServletResponse servletResponse = context.getResponse()
    servletResponse.setCharacterEncoding("UTF-8")

    OutputStream outStream = servletResponse.getOutputStream();
    InputStream is = null
    try {
        // 如果上下文中设置了responseBody，则覆盖掉原来的输出
        if (RequestContext.currentContext.responseBody != null) {
            String body = RequestContext.currentContext.responseBody
            writeResponse(new ByteArrayInputStream(body.getBytes(Charset.forName("UTF-8"))), outStream)
            return;
        }

        // 判断请求是否能接收gzip压缩
        boolean isGzipRequested = false
        final String requestEncoding = context.getRequest().getHeader(ZuulHeaders.ACCEPT_ENCODING)
        if (requestEncoding != null && requestEncoding.equals("gzip"))
            isGzipRequested = true;

        is = context.getResponseDataStream();
        InputStream inputStream = is
        if (is != null) {
            if (context.sendZuulResponse()) {
                // if origin response is gzipped, and client has not requested gzip, decompress stream
                // before sending to client
                // else, stream gzip directly to client
                // 如果原始响应是gzip压缩的，但请求来源并不接受gzip，就解压后再返回，否则直接返回gzip响应
                if (context.getResponseGZipped() && !isGzipRequested)
                    try {
                        inputStream = new GZIPInputStream(is);
                    } catch (java.util.zip.ZipException e) {
                        println("gzip expected but not received assuming unencoded response" + RequestContext.currentContext.getRequest().getRequestURL().toString())
                        inputStream = is
                    }
                else if (context.getResponseGZipped() && isGzipRequested)
                    servletResponse.setHeader(ZuulHeaders.CONTENT_ENCODING, "gzip")
                writeResponse(inputStream, outStream)
            }
        }
    } finally {
        try {
            is?.close();
            outStream.flush()
            outStream.close()
        } catch (IOException e) {
        }
    }
}

// 其它方法比较简单，省略了...
```

### Stats

负责收集统计数据，以及将filter执行过程中收集到的debug信息输出到控制台。

```java
@Override
boolean shouldFilter() {
    return true
}

@Override
Object run() {
    int status = RequestContext.getCurrentContext().getResponseStatusCode();
    StatsManager sm = StatsManager.manager
    // 收集统计数据
    sm.collectRequestStats(RequestContext.getCurrentContext().getRequest());
    sm.collectRouteStats(RequestContext.getCurrentContext().route, status);
    // 打印debug信息
    dumpRoutingDebug()
    dumpRequestDebug()
}
```





