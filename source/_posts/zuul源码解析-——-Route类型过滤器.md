---
title: zuul源码解析 —— Route类型过滤器
urlname: source/zuul/filters/route
date: 2018-12-01 14:17:48
categories:
    - source
    - zuul
tags:
---

## Route类型过滤器

Route类型的过滤器在zuul中负责对请求进行转发，zuul作为网关的最核心的功能就体现在route类型过滤器中。Netflix提供了两个route类型的过滤器：ZuulHostRequest和ZuulNFRequest，但任意请求只会满足其中一个过滤器的执行条件。

Route类型过滤器执行顺序为：ZuulNFRequest(10) -> ZuulHostRequest(100)

### ZuulHostRequest

ZuulHostRequest简单地根据host来转发请求，host的值根据上下文变量routeHost取得。

```java
// ZuulHostRequest.groovy
boolean shouldFilter() {
    // 如果routeHost不为空，则说明此次请求是转发到指定host，由此过滤器进行处理，否则由ZuulNFRequest进行处理
    return RequestContext.currentContext.getRouteHost() != null && RequestContext.currentContext.sendZuulResponse()
}

Object run() {
    HttpServletRequest request = RequestContext.currentContext.getRequest();
    // 构建请求headers，会包括原请求请求头和zuul添加的请求头
    Header[] headers = buildZuulRequestHeaders(request)
    // 获取HTTP动词，即GET/POST之类
    String verb = getVerb(request);
    InputStream requestEntity = getRequestBody(request)
    HttpClient httpclient = CLIENT.get()

    String uri = request.getRequestURI()
    if (RequestContext.currentContext.requestURI != null) {
        // 如果在之前的过滤器中为上下文添加了requestURI，则覆盖掉原uri
        uri = RequestContext.currentContext.requestURI
    }

    try {
        // 转发请求并将响应保存到上下文
        HttpResponse response = forward(httpclient, verb, uri, request, headers, requestEntity)
        setResponse(response)
    } catch (Exception e) {
        throw e;
    }
    return null
}
```

ZuulHostRequest的run方法会根据原请求和上下文构建一个新的HTTP请求，然后进行转发，接下来看看forward方法

```java
private static final AtomicReference<HttpClient> CLIENT = new AtomicReference<HttpClient>(newClient());

 HttpResponse forward(HttpClient httpclient, String verb, String uri, HttpServletRequest request, Header[] headers, InputStream requestEntity) {
    // 如果开启了debugRequest的话，这里会返回一个wrap过的requestEntity(debugRequestEntity)用于debug
    requestEntity = debug(httpclient, verb, uri, request, headers, requestEntity)

    org.apache.http.HttpHost httpHost

    httpHost = getHttpHost()

    org.apache.http.HttpRequest httpRequest;

    switch (verb) {
        case 'POST':
            httpRequest = new HttpPost(uri + getQueryString())
            InputStreamEntity entity = new InputStreamEntity(requestEntity, request.getContentLength())
            httpRequest.setEntity(entity)
            break
        case 'PUT':
            httpRequest = new HttpPut(uri + getQueryString())
            InputStreamEntity entity = new InputStreamEntity(requestEntity, request.getContentLength())
            httpRequest.setEntity(entity)
            break;
        default:
            httpRequest = new BasicHttpRequest(verb, uri + getQueryString())
    }

    try {
        httpRequest.setHeaders(headers)
        HttpResponse zuulResponse = executeHttpRequest(httpclient, httpHost, httpRequest)
        return zuulResponse
    } finally {
        // When HttpClient instance is no longer needed,
        // shut down the connection manager to ensure
        // immediate deallocation of all system resources
        // 这行被注释掉了，可能因为之前用的client不是静态变量，现在换成静态变量后就不需要在这里释放连接了
//            httpclient.getConnectionManager().shutdown();
    }
}

HttpResponse executeHttpRequest(HttpClient httpclient, HttpHost httpHost, HttpRequest httpRequest) {
    // 封装成hystrix command执行，有关hystrix的内容这里不做讨论
    HostCommand command = new HostCommand(httpclient, httpHost, httpRequest)
    command.execute();
}
```


### ZuulNFRequest

ZuulNFRequest可以将请求路由到注册到eureka server的服务中。它使用Ribbon客户端，可以将请求就当前可用实例进行负载均衡。

```java
// ZuulNFRequest.groovy
boolean shouldFilter() {
    return NFRequestContext.currentContext.getRouteHost() == null && RequestContext.currentContext.sendZuulResponse()
}

Object run() {
    NFRequestContext context = NFRequestContext.currentContext
    HttpServletRequest request = context.getRequest();

    // 构建请求headers，会包括原请求请求头和zuul添加的请求头
    MultivaluedMap<String, String> headers = buildZuulRequestHeaders(request)
    // 构建请求参数
    MultivaluedMap<String, String> params = buildZuulRequestQueryParams(request)
    Verb verb = getVerb(request);
    Object requestEntity = getRequestBody(request)
    // 通过routeVIP获取到相应的ribbon客户端
    IClient restClient = ClientFactory.getNamedClient(context.getRouteVIP());

    String uri = request.getRequestURI()
    if (context.requestURI != null) {
        uri = context.requestURI
    }
    //remove double slashes
    uri = uri.replace("//", "/")

    HttpResponse response = forward(restClient, verb, uri, headers, params, requestEntity)
    setResponse(response)
    return response
}
```

逻辑基本上与ZuulHostRequest一致，唯一不同的是使用的HTTP客户端不一样。ZuulNFRequest使用的客户端是从ClientFactory中获取的命名客户端(named client)，每个命名客户端会有一个相应的命名配置(named config)，这意味者每个客户端都可以有不同的配置。来看看相关代码

tip: 这个ClientFactory其实是属于ribbon的类，如果对ribbon有了解可以选择跳过下面的内容不看。

```java
// ClientFactory.java
public static synchronized IClient getNamedClient(String name, Class<? extends IClientConfig> configClass) {
    if (simpleClientMap.get(name) != null) {
        return simpleClientMap.get(name);
    }
    try {
        // client不存在，尝试创建一个
        return createNamedClient(name, configClass);
    } catch (ClientException e) {
        throw new RuntimeException("Unable to create client", e);
    }
}

public static synchronized IClient createNamedClient(String name, Class<? extends IClientConfig> configClass) throws ClientException {
    // 获取命名配置
    IClientConfig config = getNamedConfig(name, configClass);
    return registerClientFromProperties(name, config);
}
```

需要注意到的是在createNamedClient()中，传入的client config是一个类型，通过getNamedConfig(name, configClass)获取到当前客户端对应的命名配置。

#### 命名配置

所谓命名配置即是为每份配置起一个名字，然后在运行时就可以根据不同的名字来获取不同的配置，例如我们可以这样配置

```properties
# zuul.properties
origin.zuul.client.DeploymentContextBasedVipAddresses=ORIGIN
origin.zuul.client.Port=8080
foo.zuul.client.DeploymentContextBasedVipAddresses=FOO
foo.zuul.client.Port=8081
bar.zuul.client.DeploymentContextBasedVipAddresses=BAR
bar.zuul.client.Port=8082
```

然后构造出来的origin, foo, bar这三个客户端对应的配置分别是vip: ORIGIN, FOO, BAR; port: 8080, 8081, 8082

获取命名配置的代码如下

```java
// ClientFactory.java
public static IClientConfig getNamedConfig(String name, Class<? extends IClientConfig> clientConfigClass) {
    IClientConfig config = namedConfig.get(name);
    if (config != null) {
        return config;
    } else {
        try {
            config = (IClientConfig) clientConfigClass.newInstance();
            // 以名称前缀加载相应的配置
            config.loadProperties(name);
        } catch (Throwable e) {
            logger.error("Unable to create client config instance", e);
            return null;
        }
        config.loadProperties(name);
        IClientConfig old = namedConfig.putIfAbsent(name, config);
        if (old != null) {
            config = old;
        }
        return config;
    }
}
```





