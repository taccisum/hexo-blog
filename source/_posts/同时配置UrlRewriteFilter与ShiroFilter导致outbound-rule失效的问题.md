---
title: 同时配置UrlRewriteFilter与ShiroFilter导致outbound rule失效的问题
urlname: outbound_rule_invalid_cause_by_urlrewritefilter_and_shirofilter
date: 2017-12-21 16:10:51
categories:
  - java
  - 踩坑日记
tags:
---
## 项目环境
Spring Boot1.5.4 + Shiro1.4.0 + UrlRewriteFilter4.0.4

## 问题描述
最近在项目中同时用到[Shiro](http://shiro.apache.org/)和[UrlRewriteFilter](http://www.tuckey.org/urlrewrite/)。由于在web环境下两者都是通过配置filter来实现功能的，因此导致出现了冲突——UrlRewriteFilter的outbound rule无法正常工作。具体表现为：

在urlrewrite.xml中配置了
```xml
<urlrewrite>
    <outbound-rule>
        <from>/in-rewrite\?type=(\w+)$</from>
        <to>/in/$1.html</to>
    </outbound-rule>
</urlrewrite>
```

在java代码中定义action
```java
    @GetMapping("out-rewrite")
    @ResponseBody
    public String out(HttpServletRequest request, HttpServletResponse response) {
        return response.encodeURL("localhost:8080/in-rewrite?type=233");       //will be rewrite to 'localhost:8080/in/233.html'
    }
```

正常来说，访问这个api应该返回"localhost:8080/in/233.html"才对，但却返回了未转换前的url，即"localhost:8080/in-rewrite?type=233"。

## 问题分析
在调试源码的时候发现，同时配置了ShiroFilter和UrlRewriteFilter时，response的类型是`ShiroHttpServletResponse`，此时outbound rule是不起作用的。而把ShiroFilter的配置去掉，只留下UrlRewriteFilter时，response的类型则是`UrlRewriteWrappedResponse`，而此时outbound rule是起作用的。显然，两者在filter里面都对response进行了重新包装，而ShiroFilter执行在后，导致前者的部分功能失效。

再查看上面两个response包装类的源码：

**UrlRewriteWrappedResponse.java**
```java
...
    public String encodeURL(String s) {
        RewrittenOutboundUrl rou = this.processPreEncodeURL(s);
        if(rou == null) {
            return super.encodeURL(s);
        } else {
            if(rou.isEncode()) {
                rou.setTarget(super.encodeURL(rou.getTarget()));
            }

            return this.processPostEncodeURL(rou.getTarget()).getTarget();
        }
    }
...
```

**ShiroHttpServletResponse.java**
```java
...
    public String encodeURL(String url) {
        String absolute = toAbsolute(url);
        if (isEncodeable(absolute)) {
            // W3c spec clearly said
            if (url.equalsIgnoreCase("")) {
                url = absolute;
            }
            return toEncoded(url, request.getSession().getId());
        } else {
            return url;
        }
    }
...
```

发现两者均对`encodeUrl`、`encodeRedirectUrl`等四个方法进行了改写。Shiro的包装类执行在后，导致`UrlRewriteWrappedResponse`的相关代码没有被执行。

## 解决方案
### 重排Filter的执行顺序
让`UrlRewriteWrappedResponse`的包装过程执行在后，避免encodeUrl方法被改写。

显然这不是一个好的办法，原因如下：
1. 导致`ShiroHttpServletResponse`的方法被改写，可能引出其它未知问题，无异于拆东墙补西墙
2. 导致ShiroFilter执行在前，可能导致一些访问的url还未被重写，这次访问就被Shiro拦截了

### 自己实现encodeUrl的逻辑
让`ShiroHttpServletResponse`执行encode前先执行`UrlRewriteWrappedResponse`的encode代码。

1. 改写`ShiroHttpServletResponse`，重写其encode逻辑
```java
    public class CustomShiroHttpServletResponse extends ShiroHttpServletResponse {
        private HttpServletResponse wrapped;

        public CustomShiroHttpServletResponse(HttpServletResponse wrapped, ServletContext context, ShiroHttpServletRequest request) {
            super(wrapped, context, request);
            this.wrapped = wrapped;
        }

        @Override
        public String encodeRedirectURL(String url) {
            return super.encodeRedirectURL(wrapped.encodeRedirectURL(url));
        }

        @Override
        public String encodeRedirectUrl(String s) {
            return super.encodeRedirectUrl(wrapped.encodeRedirectUrl(s));
        }

        @Override
        public String encodeURL(String url) {
            return super.encodeURL(wrapped.encodeURL(url));
        }

        @Override
        public String encodeUrl(String s) {
            return super.encodeUrl(wrapped.encodeUrl(s));
        }
    }
```

2. 改写`ShiroFilter`，使用`CustomShiroHttpServletResponse`代替`ShiroHttpServletResponse`
```java
public class CustomSpringShiroFilter extends AbstractShiroFilter {
    protected CustomSpringShiroFilter(WebSecurityManager webSecurityManager, FilterChainResolver resolver) {
        //这段代码是copy了ShiroFilterFactoryBean$SpringShiroFilter的构造方法中的代码
        super();
        if (webSecurityManager == null) {
            throw new IllegalArgumentException("WebSecurityManager property cannot be null.");
        }
        setSecurityManager(webSecurityManager);
        if (resolver != null) {
            setFilterChainResolver(resolver);
        }
    }

    @Override
    protected ServletResponse wrapServletResponse(HttpServletResponse orig, ShiroHttpServletRequest request) {
        //使用CustomShiroHttpServletResponse代替原有的Response Wrapper
        return new CustomShiroHttpServletResponse(orig, getServletContext(), request);
    }
}
```

3. 然后改写`ShiroFilterFactoryBean`的createInstance方法，使用`CustomSpringShiroFilter`，代替原来的Filter
```java
public class CustomShiroFilterFactoryBean extends ShiroFilterFactoryBean {
    private static transient final Logger log = LoggerFactory.getLogger(ShiroFilterFactoryBean.class);

    @Override
    protected AbstractShiroFilter createInstance() throws Exception {
        //这部分代码与父类相同
        log.debug("Creating Shiro Filter instance.");

        SecurityManager securityManager = getSecurityManager();
        if (securityManager == null) {
            String msg = "SecurityManager property must be set.";
            throw new BeanInitializationException(msg);
        }

        if (!(securityManager instanceof WebSecurityManager)) {
            String msg = "The security manager does not implement the WebSecurityManager interface.";
            throw new BeanInitializationException(msg);
        }

        FilterChainManager manager = createFilterChainManager();

        PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
        chainResolver.setFilterChainManager(manager);

        //使用CustomSpringShiroFilter代替SpringShiroFilter
        return new CustomSpringShiroFilter((WebSecurityManager) securityManager, chainResolver);
    }
}
```

4. 重启项目，访问"/out-rewrite"，结果如下
![outbound rule](/images/20171221/outbound.png)
