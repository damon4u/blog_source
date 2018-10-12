---
title: IDEA插件开发（五）Service
date: 2018-10-12 16:41:50
tags: idea
categories: [idea,java]
---

IDEA平台提供`service`的概念。一个`Service`组件是单例的，使用`ServiceManager`的`getService`方法获取。
`Service`可以是一个类，也可以是一个接口，但如果是接口，必须有实现类。

<!-- more -->

有三种类型的service：
* application级别
* project级别
* module级别

与`Action`类似新建类时，可以选择模板来创建`Service`。

注册：
```xml
<extensions defaultExtensionNs="com.intellij">
  <!-- Declare the application level service -->
  <applicationService serviceInterface="Mypackage.MyApplicationService" serviceImplementation="Mypackage.MyApplicationServiceImpl" />

  <!-- Declare the project level service -->
  <projectService serviceInterface="Mypackage.MyProjectService" serviceImplementation="Mypackage.MyProjectServiceImpl" />
</extensions>
```
如果不指定`serviceInterface`，那么跟`serviceImplementation`一样。
获取：
```java
MyApplicationService applicationService = ServiceManager.getService(MyApplicationService.class);

MyProjectService projectService = ServiceManager.getService(project, MyProjectService.class);

MyModuleService moduleService = ModuleServiceManager.getService(module, MyModuleService.class);
```

参考资料：
[Plugin Services](https://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_services.html)
