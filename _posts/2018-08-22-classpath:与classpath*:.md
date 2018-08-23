---
title: classpath: 与 classpath*:
date: 2018-08-22 16:41:50
tags: spring
categories: spring
---

classpath是指WEB-INF文件夹下的classes目录。通常我们需要在Spring的容器配置文件中指定类路径下的配置文件。
本文介绍`classpath:` 与 `classpath*:`写法的区别。

<!-- more -->

通常我们会将jdbc.properties放到resources下，maven打包后放到classes目录中。
```xml
<!-- 读取JDBC配置文件 -->
<context:property-placeholder location="classpath:jdbc.properties" ignore-unresolvable="true"/>
```
或是将容器配置文件按功能划分，最后通过import，统一到一个总的配置文件中：
```xml
<import resource="classpath:database/daoContext.xml"/>
```
以上两种情况我们都是引入本项目resources中的文件，也就是直接在classes目录中。

有时候我们需要引用第三方jar包中的配置文件，此时，这些没有直接的出现在classes目录中，我们需要spring帮我们去所有的classpath找，这是就需要`classpath*:`了：
```
<import resource="classpath*:delayMessageConsumerContext.xml"/>
```

需要注意的是，用classpath*:需要遍历所有的classpath，所以加载速度是很慢，应该尽量避免使用。
