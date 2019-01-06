---
title: "Spring MVC3中的默认设置"
categories: java spring
tags: Spring
---

## 创建一个spring mvc3的项目

## 编写代码
利用下面的代码来查看spring中的内置Bean
```java
@Override
protected WebApplicationContext initWebApplicationContext() {
    WebApplicationContext ctx = super.initWebApplicationContext();
    for (Map.Entry<String, Object> ent : ctx. getBeansOfType(Object.class).entrySet()) {
        System.out.println(ent.getKey() + ":" + ent.getValue().getClass());
    }

    return ctx;
}
```

## 查看标签的作用 mvc:annotation-driven

### 添加前
```
environment: class org.springframework.web.context.support.StandardServletEnvironment
systemProperties: class java.util.Properties
systemEnvironment: class java.util.Collections$UnmodifiableMap
servletContext: class org.apache.catalina.core.ApplicationContextFacade
servletConfig: class org.apache.catalina.core.StandardWrapperFacade
contextParameters: class java.util.Collections$UnmodifiableMap
contextAttributes: class java.util.Collections$UnmodifiableMap
messageSource: class org.springframework.context.support.DelegatingMessageSource
applicationEventMulticaster: class org.springframework.context.event.SimpleApplicationEventMulticaster
lifecycleProcessor: class org.springframework.context.support.DefaultLifecycleProcessor
```

### 添加后
```
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#0:
    class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
org.springframework.format.support.FormattingConversionServiceFactoryBean#0:
    class org.springframework.format.support.DefaultFormattingConversionService
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#0:
    class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
org.springframework.web.servlet.handler.MappedInterceptor#0: class org.springframework.web.servlet.handler.MappedInterceptor
org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver#0:
    class org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver
org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver#0:
    class org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver
org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver#0:
    class org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping:
    class org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter:
    class org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter
org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter:
    class org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter
environment: class org.springframework.web.context.support.StandardServletEnvironment
systemProperties:class java.util.Properties
systemEnvironment:class java.util.Collections$UnmodifiableMap
servletContext:class org.apache.catalina.core.ApplicationContextFacade
servletConfig:class org.apache.catalina.core.StandardWrapperFacade
contextParameters:class java.util.Collections$UnmodifiableMap
contextAttributes:class java.util.Collections$UnmodifiableMap
messageSource:class org.springframework.context.support.DelegatingMessageSource
applicationEventMulticaster:class org.springframework.context.event.SimpleApplicationEventMulticaster
```


### 多出来的
这些就是`mvc:annotation-driven`添加的的
```
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping#0:
    class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping
org.springframework.format.support.FormattingConversionServiceFactoryBean#0:
    class org.springframework.format.support.DefaultFormattingConversionService
org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter#0:
    class org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter
org.springframework.web.servlet.handler.MappedInterceptor#0: class org.springframework.web.servlet.handler.MappedInterceptor
org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver#0:
    class org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver
org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver#0:
    class org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver
org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver#0:
    class org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping:
    class org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping
org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter:
    class org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter
org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter:
    class org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter
```
