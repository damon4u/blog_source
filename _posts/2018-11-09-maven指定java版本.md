---
title: maven指定java版本
date: 2018-11-08 17:41:50
tags: maven
categories: maven
---

如果使用Spring Boot搭建项目，可以指定property：
```xml
<properties>
     <java.version>1.8</java.version>
</properties>
```
注意`java.version`这个参数并不是maven提供的，而是Spring Boot特有的。

<!-- more -->

如果想单纯的使用maven特性指定，那么有两种方式，它们是等价的。
第一种，指定property：
```
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties>
```
第二种，使用插件：
```
<plugins>
    <plugin>    
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
            <source>1.8</source>
            <target>1.8</target>
        </configuration>
    </plugin>
</plugins>
```
插件中的`source`和`target`属性就是会使用`maven.compiler.source`和`maven.compiler.target`，如果你指定的话。
> source <p>
> The -source argument for the Java compiler. <p>
Default value is: 1.6. <p>
User property is: maven.compiler.source. <p>
> target <p>
> The -target argument for the Java compiler. <p>
Default value is: 1.6. <p>
User property is: maven.compiler.target.

参考：
[Specifying java version in maven - differences between properties and compiler plugin](https://stackoverflow.com/questions/38882080/specifying-java-version-in-maven-differences-between-properties-and-compiler-p)
[Setting the -source and -target of the Java Compiler](https://maven.apache.org/plugins/maven-compiler-plugin/examples/set-compiler-source-and-target.html)
