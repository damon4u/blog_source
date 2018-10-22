---
title: IDEA插件开发（七）mybatis插件之mapper解析
date: 2018-10-18 17:41:50
tags: idea
categories: [idea,java]
---

从今天开始一步一步练习mybatis插件开发。

第一步是解析mapper文件，可以实现xml中按照id属性查找resultMap或者sql片段，鼠标点击跳转。

Dom操作Xml的基本只是可以参考之前的文章：[IDEA插件开发（二）DOM操作XML文件](https://damon4u.github.io/blog/2018/10/IDEA%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91%EF%BC%88%E4%BA%8C%EF%BC%89DOM%E6%93%8D%E4%BD%9CXML%E6%96%87%E4%BB%B6.html#more)

本文直接以mybatis的mapper文件解析为例，说明Dom操作的常用写法。

<!-- more -->

使用Dom解析xml文件，首先需要按照xml包含的标签元素（DTD）编写对应的Java实体。
例如一个典型的mapper xml文件包含的标签元素为：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.demo.dao.UserDao">
    <resultMap id="userMap" type="com.demo.entity.User">
        <id column="id" property="id"/>
        <result column="user_id" property="userId"/>
        <result column="user_name" property="userName"/>
        <association property="jobInfo" javaType="com.demo.entity.Job"
                    resultMap="jobMap"/>
    </resultMap>

    <sql>

    </sql>

    <insert id="insertUser" parameterType="com.demo.entity.User">

    </insert>

    <delete id="deleteUser">

    </delete>

    <update id="updateUser">

    </update>

    <select id="getUser" resultMap="userMap">

    </select>
</mapper>
```

转化为一个Java实体：
```java
public interface Mapper extends DomElement {

    /**
     * 一次返回mapper文件中增删改查子tag，在寻找候选tag时需要
     * @return 增删改查子tag列表，每个子tag都包含id属性
     */
    @NotNull
    @SubTagsList({"insert", "update", "delete", "select"})
    List<IdDomElement> getDaoElements();

    /**
     * mapper标签包含namespace属性
     */
    @Required
    @NameValue
    @NotNull
    @Attribute("namespace")
    GenericAttributeValue<String> getNamespace();

    /**
     * 解析并返回resultMap标签列表
     */
    @NotNull
    @SubTagList("resultMap")
    List<ResultMap> getResultMaps();

    /**
     * 解析并返回parameterMap标签列表
     */
    @NotNull
    @SubTagList("parameterMap")
    List<ParameterMap> getParameterMaps();

    /**
     * 解析并返回sql标签列表
     */
    @NotNull
    @SubTagList("sql")
    List<Sql> getSqls();

    /**
     * 解析并返回insert标签列表
     */
    @NotNull
    @SubTagList("insert")
    List<Insert> getInserts();

    /**
     * 解析并返回update标签列表
     */
    @NotNull
    @SubTagList("update")
    List<Update> getUpdates();

    /**
     * 解析并返回delete标签列表
     */
    @NotNull
    @SubTagList("delete")
    List<Delete> getDeletes();

    /**
     * 解析并返回select标签列表
     */
    @NotNull
    @SubTagList("select")
    List<Select> getSelects();

}
```
可以看到，每个xml中想解析的标签，都要写一个get方法去标示。
这里解释几个常用的注解：

#### SubTag
`@SubTag`标注最多出现一次的子标签，对应的get方法返回单个实体。
例如`resultMap`标签中，最多有一个`constructor`子标签，那么需要写为：
```java
@SubTag("constructor")
Constructor getConstructor();
```

#### SubTagList
`@SubTagList`标注可以出现多次的子标签集合，对应的get方法返回实体列表。
例如一个`mapper`标签中，可以有多个`resultMap`子标签，那么需要写为：
```java
@SubTagList("resultMap")
List<ResultMap> getResultMaps();
```

#### SubTagsList
`@SubTagsList`标注的方法，返回多个集合的全集，对应的方法返回实体列表。
通常用于返回一个标签中的多个子标签集合。
例如，我们要定义一个方法，返回mapper中出现的所有增删改查子标签集合，那么可以写为：
```java
@SubTagsList({"insert", "update", "delete", "select"})
List<IdDomElement> getDaoElements();
```

#### NameValue
`@NameValue`的作用：使用这个注解标注`Element`中的一个方法，这个方法只能返回`String`或者`GenericValue`，在使用`ElementPresentationManager#getElementName(Object)`获取当前`Element`的名字标示时，返回这个方法的返回值，或者是直接的`String`，或者是`GenericValue`，那么返回`GenericValue`的`getStringValue`方法结果。

#### Attribute
`@Attribute`标注的方法，在xml中是以标签属性的形式出现的，而非标签。例如mapper中的namespace属性。
```xml
<mapper namespace="com.demo.dao.UserDao">
</mapper>
```
那么对应的方法为：
```java
@Attribute("namespace")
GenericAttributeValue<String> getNamespace();
```

上面给出的`Mapper`接口继承了`DomElement`接口，这样就成为了一个可以DOM被解析的xml元素。
`Mapper`接口中出现的实体也都是需要自己定义的，如`ResultMap`,`Insert`等。

下面开始写单个实体，对mapper中的子元素进行解析。

### 一个包含id属性的基础标签
很多标签都包含`id`属性，例如`resultMap`，`insert`等，那么我们先定义一个基本标签：
```java
/**
 * Description:
 * 定义一个基本dom元素，包含id属性
 *
 * @author damon4u
 * @version 2018-10-18 15:38
 */
public interface IdDomElement extends DomElement {

    /**
     * 必须包含一个id属性
     * Required注解标示必须要有该属性
     * NameValue注解该方法，那么外部获取IdDomElement的标示时，返回该方法的结果
     * Attribute注解标示xml属性
     * @return 返回GenericAttributeValue而不是String可以方便重命名
     */
    @Required
    @NameValue
    @Attribute("id")
    GenericAttributeValue<String> getId();
}
```

### 动态查询语句元素
mapper中允许使用`where`，`set`，`if`等动态查询语句拼接查询条件，那么可以抽取出来一个动态查询语句元素：
```java
/**
 * Description:
 * mapper中的动态查询语句
 * http://www.mybatis.org/mybatis-3/dynamic-sql.html
 *
 * @author damon4u
 * @version 2018-10-18 16:55
 */
public interface DynamicQueryableDomElement extends DomElement {

    @NotNull
    @SubTagList("include")
    List<Include> getIncludes();

    @NotNull
    @SubTagList("trim")
    List<Trim> getTrims();

    @NotNull
    @SubTagList("where")
    List<Where> getWheres();

    @NotNull
    @SubTagList("set")
    List<Set> getSets();

    @NotNull
    @SubTagList("foreach")
    List<Foreach> getForeachs();

    @NotNull
    @SubTagList("choose")
    List<Choose> getChooses();

    @NotNull
    @SubTagList("if")
    List<If> getIfs();

    @SubTagList("bind")
    List<Bind> getBinds();
}
```
其中`Trim`，`Where`等子标签又可以包含动态查询语句，因此，可以写为类似递归的形式：
```java
public interface Where extends DynamicQueryableDomElement {

}
```
其中`Include`标签要特殊一些，根据mybatis的规则，该标签用来引用sql代码片段。
所谓sql代码片段，是一段可复用的动态查询语句集合，因此我们先定义`Sql`实体：
```java
public interface Sql extends DynamicQueryableDomElement, IdDomElement {

}
```
可以看到它继承了`DynamicQueryableDomElement`和`IdDomElement`，代表着它包含了动态查询语句，带有`id`属性。
下面就可以定义`include`实体了：
```java
public interface Include extends DomElement {

    /**
     * include引用标签id
     * @return 引用标签
     */
    @Attribute("refid")
    GenericAttributeValue<Sql> getRefId();
}
```
注意返回值泛型参数为`Sql`实体，这样在xml编写时，IDEA可以自动提供代码补全、鼠标点击跳转引用等功能。
