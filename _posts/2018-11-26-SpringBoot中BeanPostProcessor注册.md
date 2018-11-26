---
title: SpringBoot中BeanFactoryPostProcessor注册
date: 2018-11-26 16:41:50
tags: spring
categories: spring
---

在使用SpringBoot集成shiro时，遇到过一个问题，shiro需要注册一个`LifecycleBeanPostProcessor`，用来调用内部的`init()`和`destroy()`方法。
一般在Configuration中注册：
```java
@Configuration
public class SpringServiceConfig {

    @Autowired
    private SpringDataSourceConfig springDataSourceConfig;

    private SpringRedisConfig springRedisConfig;

    @Autowired
    public SpringServiceConfig(SpringRedisConfig springRedisConfig) {
        this.springRedisConfig = springRedisConfig;
    }

    //sessionManager securityManager realm...

    @Bean
    public static LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
        return new LifecycleBeanPostProcessor();
    }

    @Bean
    @DependsOn({"lifecycleBeanPostProcessor"})
    public DefaultAdvisorAutoProxyCreator advisorAutoProxyCreator() {
        DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();
        defaultAdvisorAutoProxyCreator.setProxyTargetClass(true);
        return defaultAdvisorAutoProxyCreator;
    }
}
```

<!-- more -->

在这个`Configuration`中，使用`@Autowired`注入会失败！
如果是在属性上使用：
```java
@Autowired
private SpringDataSourceConfig springDataSourceConfig;
```
使用时发现springDataSourceConfig为空；
如果用构造器注入：
```java
private SpringRedisConfig springRedisConfig;

@Autowired
public SpringServiceConfig(SpringRedisConfig springRedisConfig) {
    this.springRedisConfig = springRedisConfig;
}
```
会报错：
```
No default constructor found;
```

网上搜索后找到答案，原因是`LifecycleBeanPostProcessor`集成了`BeanPostProcessor`，而在Spring声明周期中，`BeanPostProcessor`和`BeanFactoryPostProcessor`的初始化是优先于bean的初始化的：

![](/images/spring0.png)

![](/images/spring1.png)

因此，如果一个`@Configuration`标注的类中返回了`BeanPostProcessor`或者`BeanFactoryPostProcessor`，那么无法使用`@Autowired`进行注入。
`@Bean`注解的javadoc中也有说明：
> BeanFactoryPostProcessor-returning `@Bean` methods <br>
Special consideration must be taken for `@Bean` methods that return Spring __BeanFactoryPostProcessor (BFPP)__ types. Because BFPP objects must be instantiated very early in the container lifecycle, they can interfere(干涉) with processing of annotations such as `@Autowired`, `@Value`, and `@PostConstruct` within `@Configuration` classes. To avoid these lifecycle issues, mark BFPP-returning `@Bean` methods as __static__. For example:<br>
```
       @Bean
       public static PropertyPlaceholderConfigurer ppc() {
           // instantiate, configure and return ppc ...
       }
```
<br>
By marking this method as static, it can be invoked without causing instantiation of its declaring `@Configuration` class, thus avoiding the above-mentioned lifecycle conflicts. Note however that static `@Bean` methods will not be enhanced for scoping and AOP semantics as mentioned above. This works out in BFPP cases, as they are not typically referenced by other `@Bean` methods. As a reminder, a WARN-level log message will be issued for any non-static `@Bean` methods having a return type assignable to BeanFactoryPostProcessor.

解决的方案javadoc中也说了，就是给返回`BeanPostProcessor`或者`BeanFactoryPostProcessor`的方法使用static修饰。
这样，调用该方法时，不需要实例化外层的容器。



参考：
[Spring Java based configuration with static method](https://stackoverflow.com/questions/31763257/spring-java-based-configuration-with-static-method)
[Spring Bean的生命周期](https://www.cnblogs.com/zrtqsk/p/3735273.html)
