---
title: 声明式http客户端openfeign的研究与应用
urlname: openfeign_research_and_application
date: 2018-06-21 05:54:42
categories:
tags:
---


# 简介
## openfeign
openfeign，简称feign，是netflix开源的技术栈之一。
feign的主旨是使得编写java http客户端更容易。为了贯彻这个理念，feign采用了通过处理注解来自动生成请求的方式（官方称呼为`声明式`、`模板化`）。因此，基于feign编写的http客户端画风看起来是这样的

```java
interface Bank {
  @RequestLine("POST /account/{id}")
  Account getAccountInfo(@Param("id") String id);
}
```

然后通过一系列操作可以为Bank接口在运行时自动生成对应的实现。通过这个实现我们就可以在java中像调用一个本地方法一样完成一次http请求，大大减少了编码成本，同时提高了代码可读性。

## spring-cloud-openfeign
spring-cloud-feign基于自动配置功能（autoconfiguration），为spring boot应用提供openfiegn的集成：

  - 支持通过`JAX-RS`或`Spring MVC annotations`来构建feign client
  - 自动使用Spring MVC的`HttpMessageConverters`来完成序列化反序列化
  - 集成了`ribbon`和`hystrix`，只要在项目中引入相关的依赖即可立即使用
  - 无需任何配置，开箱即用，秉承了spring-boot一贯的作风。

> [官方文档参考](https://github.com/spring-cloud/spring-cloud-openfeign/blob/master/docs/src/main/asciidoc/spring-cloud-openfeign.adoc)


# 常见问题及解决方案

## 处理响应数据的通用部分
api响应数据格式，一般而言除实际的业务数据外是固定的格式，其中包括了对此次请求的处理信息描述。对于这部分数据，应由feign进行统一处理。

### 解决方案

feignclient是通过Decoder来对请求响应进行处理的，因此可以使用自定义的Decoder来处理响应数据的通用部分。

例如可以定义一个`UnwrapRestfulApiResponseSpringDecoder`来对`RestfulApiResponse`进行自动拆包操作，并通过`@FeignClient`的`configuration`属性配置其使用的Decoder。

```java
@FeignClient(name = AuthCenter.SERVICE_NAME, path = RMIPath.USER, configuration = TenantUserClient.Configuration.class)
public interface TenantUserClient {
    class Configuration {
        @Autowired
        private ObjectFactory<HttpMessageConverters> messageConverters;
 
        @Bean
        public Decoder feignDecoder() {
            return new ResponseEntityDecoder(new UnwrapRestfulApiResponseSpringDecoder(this.messageConverters));
        }
    }
}
```

## 本地调用需要注册到注册中心导致服务不可用的问题

在实际开发过程中，有时会出现需要在本地调用远程服务来进行调试的情况，此时我们需要将本地的应用注册到注册中心。而一旦这样做，会导致该服务存在多个节点（一个远程一个本地），从而导致依赖该服务的应用软负载均衡负载到本地节点时会失败。

### 解决方案

#### 1. 不通过注册中心获取信息，使用直接指定url的方式调用（✘不推荐）

**参考代码**
```java
@FeignClient(name = "sysinfo-service" ,url = "http://49.4.7.72", path = "/")
public interface SysinfoClient {
}
```

此方案优点是简单易用，但缺点也很明显：

  1. 在应用实际上线时需要将调用方式修改为通过注册中心调用，否则client将无法进行负载均衡，因此需要频繁改动代码
  2. 在特定环境下，即使忘记切换调用方式有时也不影响client的使用，会将问题隐藏，埋下隐患
  3. 并没有与其它服务真正地解耦，一旦依赖的服务故障就会导致本地开发无法进行

由于存在上述缺点，现一般**不推荐**使用该方案。

#### 2. 使用打桩的方式与远程服务解耦（✔推荐）

大致思路为，通过Spring的`@Profile`注解，在不同的环境为应用注册不同的client bean（例如在本地环境使用Hard Code bean，而在非本地环境则使用远程调用的client bean）。

**参考代码**
**ClientConfiguration.java**
```java
@EnableAuthClient
@EnableFeignClients(basePackages = "com.cheegu.icm.biz.foo.client")
@Configuration
@Profile({SpringProfiles.DEV, SpringProfiles.TEST, SpringProfiles.PROD})
public class ClientConfiguration {
}
```
**MockClientConfiguration.java**
```java
@Configuration
@Profile(SpringProfiles.LOCAL)
public class MockClientConfiguration {
    @Bean
    public SysinfoClient sysinfoClient() {
        return new SysinfoClient() {
            @Override
            public UserInfoDto user() {
                return new UserInfoDto();
            }
 
            @Override
            public TableData<UserRowDto> users() {
                return TableData.empty();
            }
        };
    }
 
    @Bean
    public TenantUserClient tenantUserClient() {
        return new TenantUserClient() {
            @Override
            public List<ResourceInfoDto> listMyAlResource() {
                return new ArrayList<>();
            }
        };
    }
}
```


## 如何在调用时带上请求头
在使用客户端进行调用时，有时会需要带上某些请求头（例如认证token），但又不希望在代码中手动去做这些操作。

### 解决方案
可以通过实现feign提供的`RequestInterceptor`接口来对请求进行切面处理。这样每次通过client发起请求时都会由feign interceptor进行拦截，为请求添加headers。

**参考代码**
```java
@Configuration
public class AppConfiguration implements EnvironmentAware {
    private Environment environment;
 
    @Bean
    public AuthorizationInfoForwardingInterceptor authorizationInfoForwardingInterceptor() {
        //这个interceptor会为请求自动带上认证token
        return new AuthorizationInfoForwardingInterceptor(environment.getProperty("spring.application.name"));
    }
 
    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```

## 使用MultipartFile类型作为请求参数

有时对外提供的接口调用可能需要传送文件，此时需要对`MultipartFile`类型的参数进行处理

### 解决方案

**待补充**




