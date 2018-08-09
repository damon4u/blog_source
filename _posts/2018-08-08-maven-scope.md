---
title: maven scope和optional
date: 2018-08-08 17:41:50
tags: maven
categories: maven
---

参考Maven Doc [Introduction to the Dependency Mechanism](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)

### 1.Transitive Dependencies（依赖传递性）
依赖传递这种特性可以帮助我们自动引入依赖包所需要的类，递归的。

* 依赖机制：当项目中的依赖有多个版本（间接依赖）时，maven会使用“最短引用原则”决定最终使用哪个版本。如果引用树深度一样，那么会使用先声明的那个。
* dependencyManagement：直接管理项目中依赖的版本，直接依赖或者间接依赖。
* scope：这个字段允许指定依赖的作用范围。
* exclusions：如果项目x依赖项目y，项目y依赖项目z，那么项目x的拥有者可以显示的排除项目z。
* optional：如果项目y依赖项目z，项目y可以将对z的引用标识为optional，这样，当项目x引用项目y时，项目x就不会引入项目z。__可以将optional理解为默认excluded。__

### 2.Dependency Scope（依赖作用域）

首先说一下classpath分类：
* compile classpath
* test classpath
* runtime classpath

#### 2.1 complie
默认就是compile。compile表示被依赖项目需要参与当前项目的编译，当然后续的测试，运行周期也参与其中，是一个比较强的依赖。打包的时候通常需要包含进去。

#### 2.2 provided
provided意味着打包的时候可以不用包进去，别的设施(Web Container)会提供。事实上该依赖理论上可以参与编译，测试，运行等周期。相当于compile，但是在打包阶段做了exclude的动作。
例如servlet，tomcat等web容易会提供。还有lombok，它在运行期不需要。
```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>${servletapi.version}</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>${lombok.version}</version>
    <scope>provided</scope>
</dependency>
```

#### 2.3 runtime
runtime表示被依赖项目无需参与项目的编译，不过后期的测试和运行周期需要其参与。
例如slf4j，运行期依赖。
```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jcl-over-slf4j</artifactId>
    <version>${slf4j.version}</version>
    <scope>runtime</scope>
</dependency>
```

#### 2.4 test
scope为test表示依赖项目仅仅参与测试相关的工作，包括测试代码的编译，执行。
比较典型的如junit。
```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>${junit.version}</version>
    <scope>test</scope>
</dependency>
```

#### 2.5 system
从参与度来说，也provided相同，不过被依赖项不会从maven仓库抓，而是从本地文件系统拿，一定需要配合`systemPath`属性使用。
