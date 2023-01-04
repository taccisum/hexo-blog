---
title: 记一次 springfox 扩展之旅 —— 动态 api 文档
urlname: 记一次 springfox 扩展之旅 —— 动态 api 文档
date: 2022-03-25 15:44:05
categories:
tags:
---

出于 [Pigeon](https://github.com/pigeon-cp/pigeon) 项目的需要，我需要把通过插件支持的扩展信息在 swagger api 文档中动态展示出来。

项目使用的是 springfox 框架生成的 swagger 文档，众所周知 springfox 是注解型的文档框架，而且文档内容是直接写死的，不支持动态化，估摸着是需要自己扩展了。

按照惯例，先了解代码架构（由于以前都是使用为主，没有深入了解，也不知道有哪些拓展点）。发现 springfox 其实是在应用启动时 scan 带有 swagger 注解的类和字段，生成一个叫 `Documentation` 的类，然后通过 mapper 转换成相应的 swagger model（v1.2 or v2）。

可见，我们要分析的 code 可以缩小到 Docuementation 生成的过程，主要关注 @ApiParam, @ApiPropertyModel 这几个注解的解析步骤。

继续深入，发现 Documentation 的生成是通过 DocumentationPlugin 来做的，而调度 plugins 则是在 DocumentationPluginsBootstrapper 中。DocumentationPluginsBootstrapper 又是从哪里被调用的呢？没错，是通过 SpringfoxWebMvcConfiguration scan package `springfox.documentation.spring.web.plugins` 时被加载到 spring context 中的，由于这是一个 SmartLifecycle，因此会自动执行 `#start` 方法。在这个方法中会 foreach plugins，通过 plugin 得到的文档 context（context 中包含了 api 信息，而这些信息其实正是 spring mvc 维护着的 request handlers），再经由 scanner scan 整个 context 生成 Documentation 后加入到 DocumentationCache 中，后续通过 `/api-v2/docs` 接口访问时就是直接从 cache 中去获取到 Documentation 的。

上面提到的 plugins 又是什么呢？这个概念大家可能比较陌生，但只要是用过 springfox 的小伙伴，对 `Docket` 这个类肯定不陌生。没错，Docket 其实也是实现了 DocumentationPlugin 的一个类，根据官方的描述

> Docket is a builder which is intended to be the primary interface into the Springfox framework.
> Provides sensible defaults and convenience methods for configuration.

回归主题，DocumentationPluginsBootstrapper 使用的 scanner 是 ApiDocumentationScanner。追踪源码发现，这个 scanner 中又细分为 ApiListingReferenceScanner（扫描引用 model 的）ApiListingScanner （扫描 apis 的），后者正是关键类。继续深入，在 ApiListingScanner -> ApiDescriptionReader -> ApiOperationReader 中发现，Operation（即维护 api 的路径、参数等具体描述的类）正是在其中通过 20+ OperationBuilderPlugin 组成的 filter 器链共同构建而成。根据名称大概筛选了一下，最终锁定了 OperationParameterReader 这个类。该类在处理参数时又会分成需要 expand 的（@RequestBody 之流）和不需要 expand 的（@ApiParam 直接标注的参数）。

先来看不需要 expand 的，最终会交由 ParameterBuilderPlugin 链处理，因此我简单写了一个扩展
```java
@Component
public class PigeonExtendParamBuilder implements ParameterBuilderPlugin {
    @Override
    public void apply(ParameterContext context) {
        if (context.resolvedMethodParameter().getParameterType().isInstanceOf(String.class)) {
            context.parameterBuilder()
                    .allowableValues(new AllowableListValues(Lists.newArrayList("MAIL", "SMS"), "String"));
        }
    }

    @Override
    public boolean supports(DocumentationType documentationType) {
        return true;
    }
}
```

跑下程序，果然所有 @ApiParam 标注的 string 类型参数都被加上了 Available values : MAIL, SMS 限制。但在 body 定义里面的参数没有效果，一开始推测是跟上述的 expand 概念有关，但后来发现不是，而是这类参数最终会指向一个 ModelRef，因此应该是与 Model 的解析有关。那 Model 的解析是在什么时候呢，同样是在 `ApiListingScanner#scan` 方法中，交由 ApiModelReader 完成。 ApiModelReader 会通过由 Spring MVC 维护的 RequestMapping 信息提取 model 信息（例如 @RequestBody, @Requestpart 之流），然后交给  `ModelProvied#modelFor` 转换成 springfox 的 Model 对象，此 Model 对象即是 ModelRef 引用的 Model。

在 `#modelFor` 中，会通过反射解析出 Model 对应的 class 中的所有字段信息（称为 ModelProperty），而这个过程则是交给 `ModelPropertiesProvider#propertiesFor` 完成的，其中细节比较多，但总的来说，最终都是交由 `ModelPropertyBuilderPlugin` 链来处理的，这个类也正是我们要找的扩展点。后续简单验证了一下，确实是有效的。

最终效果：TODO


