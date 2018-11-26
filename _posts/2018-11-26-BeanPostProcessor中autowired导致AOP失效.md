---
title: BeanPostProcessor中autowired导致AOP失效
date: 2018-11-26 17:41:50
tags: spring
categories: spring
---

在使用SpringBoot集成shiro时，遇到过一个问题，在启动时，shiroFilter中使用的bean都报WARN异常：
```
...
trationDelegate$BeanPostProcessorChecker : Bean 'adminService' of type [com.mypackage.service.AdminService] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
trationDelegate$BeanPostProcessorChecker : Bean 'sessionManager' of type [org.apache.shiro.web.session.mgt.DefaultWebSessionManager] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
trationDelegate$BeanPostProcessorChecker : Bean 'securityManager' of type [org.apache.shiro.web.mgt.DefaultWebSecurityManager] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
...
```

<!-- more -->

其中shiroFilter使用一般的配置：
```java
@Bean
public ShiroFilterFactoryBean shiroFilter() {
    ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
    shiroFilterFactoryBean.setSecurityManager(securityManager());
    shiroFilterFactoryBean.setLoginUrl("/login");
    shiroFilterFactoryBean.setSuccessUrl("/");
    Map<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
    filterChainDefinitionMap.put("/login", "authc");
    filterChainDefinitionMap.put("/**", "user");
    shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
    return shiroFilterFactoryBean;
}

@Bean
public DefaultWebSecurityManager securityManager() {
    DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
    securityManager.setRealm(shiroDbRealm());
    securityManager.setCacheManager(shiroRedisCacheManager());
    securityManager.setSessionManager(sessionManager());
    return securityManager;
}

// shiroDbRealm，shiroRedisCacheManager，sessionManager 等
```

shiroDbRealm使用`@autowired`注入了`AdminService`，它使用Spring AOP实现了动态数据源切换，根据WARN日志提示，该bean不能被aop增强，测试发现动态数据源确实不成功切换。

这是为什么呢？

搜索到答案是，`ShiroFilterFactoryBean`实现了`BeanPostProcessor`，而Spring AOP是用`BeanPostProcessor`实现的，所以其他`BeanPostProcessor`及其引用的bean都不能被aop增强。
> Classes that implement the `BeanPostProcessor` interface are special, and so they are treated differently by the container. All `BeanPostProcessors` and their __directly referenced beans__ will be instantiated on startup, as part of the special startup phase of the `ApplicationContext`, then all those `BeanPostProcessors` will be registered in a sorted fashion - and applied to all further beans. Since AOP auto-proxying is implemented as a `BeanPostProcessor` itself, no `BeanPostProcessors` or directly referenced beans are eligible for auto-proxying (and thus will not have aspects 'woven' into them. <br>
For any such bean, you should see an info log message: “Bean 'foo' is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)”.

但是realm中一定要使用数据库查询进行认证和鉴权，我这里改成了使用一个实现了`ApplicationContextAware`的`SpringContextUtil`中拿。




参考：
[Tracking down cause of Spring's “not eligible for auto-proxying”](https://stackoverflow.com/questions/1201726/tracking-down-cause-of-springs-not-eligible-for-auto-proxying)
