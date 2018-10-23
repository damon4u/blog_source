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

#### 一个包含id属性的基础标签
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

#### 动态查询语句元素
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
其他子标签与`Where`类似，不多写。
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

#### 带有请求参数的元素
增删改查元素都可以配置请求参数，包含`parameterType`指定一个实体类，那么我们抽取一个元素：
```java
public interface ParameteredDynamicQueryableDomElement extends DynamicQueryableDomElement, IdDomElement {

    @NotNull
    @Attribute("parameterType")
    // TODO: converter
    GenericAttributeValue<PsiClass> getParameterType();
}
```
其中`parameterType`的返回值泛型类型为`PsiClass`，这样能映射到一个具体的实体类上。

#### 增删改查标签
有了`ParameteredDynamicQueryableDomElement`，就可以在此基础上继续丰富，定义增删改查标签。

#### Insert
```java
public interface Insert extends ParameteredDynamicQueryableDomElement {

    @SubTagList("selectKey")
    List<SelectKey> getSelectKey();
}

/**
 * Description:
 * 插入语句中的selectkey标签，用来生成主键
 *
 * @author damon4u
 * @version 2018-10-18 17:32
 */
public interface SelectKey extends DomElement {

    @NotNull
    @Attribute("resultType")
    // TODO: converter
    GenericAttributeValue<PsiClass> getResultType();
}
```

#### Delete
```java
public interface Delete extends ParameteredDynamicQueryableDomElement {

}
```

#### Update
```java
public interface Update extends ParameteredDynamicQueryableDomElement {

}
```

#### Select
```java
public interface Select extends ParameteredDynamicQueryableDomElement, ResultMapAttributeDomElement {

    @NotNull
    @Attribute("resultType")
    // TODO: converter
    GenericAttributeValue<PsiClass> getResultType();
}
```
其中返回`resultType`好理解，指定一个实体类，而mybatis规定select标签是可以返回`resultMap`的。
这里提现在`ResultMapAttributeDomElement`：
```java
public interface ResultMapAttributeDomElement extends DomElement {

    @NotNull
    @Attribute("resultMap")
    // TODO: converter
    GenericAttributeValue<ResultMap> getResultMap();
}
```
接下来重点介绍这个相对复杂的`resultMap`。

#### resultMap

`resultMap`元素有很多子元素和一个值得讨论的结构。
* `constructor` - 用于在实例化类时，注入结果到构造方法中
  * `idArg` - ID 参数;标记出作为 ID 的结果可以帮助提高整体性能
  * `arg` - 将被注入到构造方法的一个普通结果
* `id` – 一个 ID 结果;标记出作为 ID 的结果可以帮助提高整体性能
* `result` – 注入到字段或`JavaBean`属性的普通结果
* `association` – 一个复杂类型的关联;许多结果将包装成这种类型
  * 嵌套结果映射 – associations are resultMaps themselves, or can refer to one
* `collection` – 一个复杂类型的集合
  * 嵌套结果映射 – collections are resultMaps themselves, or can refer to one
* `discriminator` – 使用结果值来决定使用哪个`resultMap`
  * `case` – 基于某些值的结果映射
    * 嵌套结果映射 – 一个 case 也是一个映射它本身的结果,因此可以包含很多相 同的元素，或者它可以参照一个外部的 resultMap。

首先，将`resultMap`中的基础元素抽取出来，他们可能直接出现在`resultMap`标签中，也可能出现在`association`等子标签中：
```java
/**
 * Description:
 * mapper中的resultMap标签内元素基本属性
 *
 * @author damon4u
 * @version 2018-10-18 17:47
 */
public interface ResultMapBaseDomElement extends DomElement {

    @SubTag("constructor")
    Constructor getConstructor();

    @SubTagList("id")
    List<Id> getIds();

    @SubTagList("result")
    List<Result> getResults();

    @SubTagList("association")
    List<Association> getAssociations();

    @SubTagList("collection")
    List<Collection> getCollections();

    @SubTag("discriminator")
    Discriminator getDiscriminator();

}
```
其中`id`和`result`子标签功能类似，用来做数据库字段与实体字段映射，包含`property`属性，那么我们将`property`属性抽取出来：
```java
/**
 * Description:
 * 包含property属性的元素
 *
 * @author damon4u
 * @version 2018-10-18 17:52
 */
public interface PropertyAttributeDomElement extends DomElement {

    @Attribute("property")
    @Convert(PropertyConverter.class)
    GenericAttributeValue<XmlAttributeValue> getProperty();
}
```
然后让`id`和`result`去继承它：
```java
public interface Id extends PropertyAttributeDomElement {

}

public interface Result extends PropertyAttributeDomElement {

}
```
而`association`和`collection`子标签功能会多一些，内部还可能包含一个嵌套的`resultMap`，因此会继承多一些，当然，它们会包含自己独有的属性：
```java
public interface Association extends ResultMapBaseDomElement, ResultMapAttributeDomElement, PropertyAttributeDomElement {

    @NotNull
    @Attribute("javaType")
    // TODO: converter
    GenericAttributeValue<PsiClass> getJavaType();
}

public interface Collection extends ResultMapBaseDomElement, ResultMapAttributeDomElement, PropertyAttributeDomElement {

    @NotNull
    @Attribute("ofType")
    // TODO: converter
    GenericAttributeValue<PsiClass> getOfType();
}
```

#### 属性转换器PropertyConverter
重新把`property`属性请回来，因为我们要讲它的转换器：
```java
/**
 * Description:
 * 包含property属性的元素
 *
 * @author damon4u
 * @version 2018-10-18 17:52
 */
public interface PropertyAttributeDomElement extends DomElement {

    @Attribute("property")
    @Convert(PropertyConverter.class)
    GenericAttributeValue<XmlAttributeValue> getProperty();
}
```
首先说目的，加这个转换器，是为了让`resultMap`中的`property`属性具有鼠标点击跳转操作，即为属性值创建引用到实体上。
假如我们有这样一个`User`实体：
```java
public class User {

    private Long id;

    private String userName;

    private Integer age;

    private User son;

}
```
我们可能会有如下`resultMap`：
```xml
<resultMap id="userResultMap" type="com.damon4u.demo.domain.User">
    <id column="id" property="id"/>
    <result column="user_name" property="userName"/>
    <result column="age" property="age"/>
    <result column="son_id" property="son.id"/>
</resultMap>
```
按照mybatis规定，除了可以映射基本类型和简单类型，还可以映射带引用嵌套层级的属性，如`son.id`。

首先创建个`MapperUtils`，写一个方法，用来获取当前`property`所属标签对应哪个实体类：
```java
public final class MapperUtils {

    private MapperUtils() {
        throw new UnsupportedOperationException();
    }

    /**
     * 获取property从属的类型
     *
     * @param attributeValue property属性
     * @return
     */
    public static Optional<PsiClass> getPropertyClazz(XmlAttributeValue attributeValue) {
        DomElement domElement = DomUtil.getDomElement(attributeValue);
        if (null == domElement) {
            return Optional.empty();
        }
        // collection标签下的property，那么类型为ofType指定
        Collection collection = DomUtil.getParentOfType(domElement, Collection.class, true);
        if (null != collection && isNotWithinSameTag(collection, attributeValue)) {
            return Optional.ofNullable(collection.getOfType().getValue());
        }
        // association标签下的property，那么类型为javaType指定
        Association association = DomUtil.getParentOfType(domElement, Association.class, true);
        if (null != association && isNotWithinSameTag(association, attributeValue)) {
            return Optional.ofNullable(association.getJavaType().getValue());
        }
        // resultMap标签下的property，那么类型为type指定
        ResultMap resultMap = DomUtil.getParentOfType(domElement, ResultMap.class, true);
        if (null != resultMap && isNotWithinSameTag(resultMap, attributeValue)) {
            return Optional.ofNullable(resultMap.getType().getValue());
        }
        return Optional.empty();

    }

    private static boolean isNotWithinSameTag(@NotNull DomElement domElement, @NotNull XmlElement xmlElement) {
        XmlTag xmlTag = PsiTreeUtil.getParentOfType(xmlElement, XmlTag.class);
        return !domElement.getXmlTag().equals(xmlTag);
    }
}
```

接下来还需要一个工具类`JavaUtils`，用来从类中寻找成员变量（非static、非final）的：
```java
public final class JavaUtils {

    private JavaUtils() {
        throw new UnsupportedOperationException();
    }

    /**
     * 从类中查找成员变量
     *
     * @param clazz        类
     * @param propertyName 属性名称
     * @return
     */
    @NotNull
    public static Optional<PsiField> findSettablePsiField(@NotNull final PsiClass clazz,
                                                          @Nullable final String propertyName) {
        final PsiField field = PropertyUtil.findPropertyField(clazz, propertyName, false);
        return Optional.ofNullable(field);
    }

    /**
     * 获取类中所有成员变量
     *
     * @param clazz 类
     * @return
     */
    @NotNull
    public static PsiField[] findSettablePsiFields(final @NotNull PsiClass clazz) {
        final PsiField[] allFields = clazz.getAllFields();
        final List<PsiField> settableFields = new ArrayList<>(allFields.length);
        for (final PsiField field : allFields) {
            final PsiModifierList modifierList = field.getModifierList();
            if (modifierList != null &&
                    (modifierList.hasModifierProperty(PsiModifier.STATIC)
                            || modifierList.hasModifierProperty(PsiModifier.FINAL))) {
                continue;
            }
            settableFields.add(field);
        }
        return settableFields.toArray(new PsiField[0]);
    }
}
```

有了工具类，就可以写解析器了，将xml属性解析到实体类字段：
```java
/**
 * Description:
 * <p>
 * 将xml属性解析到实体类字段
 *
 * @author damon4u
 * @version 2018-10-19 11:46
 */
class PsiFiledReferenceSetResolver {

    /**
     * 嵌套引用分隔符
     */
    private static final Splitter SPLITTER = Splitter.on(String.valueOf(ReferenceSetBase.DOT_SEPARATOR));

    /**
     * 当前属性元素
     */
    private XmlAttributeValue element;

    /**
     * 属性全名可能包含引用，例如user.name，那么按照点号分割，保存所有字段名称
     */
    private List<String> fieldNameWithReferenceList;

    PsiFiledReferenceSetResolver(@NotNull XmlAttributeValue element) {
        this.element = element;
        // 属性全名，可能包含引用，如user.name
        String wholeFiledName = element.getValue() != null ? element.getValue() : "";
        this.fieldNameWithReferenceList = Lists.newArrayList(SPLITTER.split(wholeFiledName));
    }

    /**
     * 将xml属性解析到实体类字段
     *
     * @param index 按照点号分割后，属性位于哪一级，index为索引值
     *              例如user.name，那么idea会为user和name分别创建引用，都可以鼠标点击跳转
     *              user的index为0，name的index为1
     *              这个值决定解析层级深度
     */
    final Optional<? extends PsiElement> resolve(int index) {
        // 先获取一级属性
        // 对于简单属性"name"，那么就是这个属性
        // 对于包含引用的情况"user.name"，那么先找到一级属性user
        Optional<PsiField> firstLevelElement = getFirstLevelElement(Iterables.getFirst(fieldNameWithReferenceList, null));
        return firstLevelElement.isPresent() ?
                (fieldNameWithReferenceList.size() > 1 ? parseNextLevelElement(firstLevelElement, index) : firstLevelElement)
                : Optional.empty();
    }

    /**
     * 获取一级属性
     * 对于简单属性"name"，那么就是这个属性
     * 对于包含引用的情况"user.name"，那么先找到一级属性user
     *
     * @param firstLevelFieldName 第一层级的属性名，可能是一个引用，也可能是基本类型
     * @return
     */
    @NotNull
    private Optional<PsiField> getFirstLevelElement(@Nullable String firstLevelFieldName) {
        Optional<PsiClass> clazz = MapperUtils.getPropertyClazz(element);
        return clazz.flatMap(psiClass -> JavaUtils.findSettablePsiField(psiClass, firstLevelFieldName));
    }

    /**
     * 按照点号逐层解析，前面层级为引用，去引用中解析下一层字段
     *
     * @param maxLevelIndex 最大解析层级深度，比如"user.name"，那么如果是要为user建立引用，maxLevelIndex为1；
     *                      如果要为name建立引用，那么maxLevelIndex为2
     */
    private Optional<PsiField> parseNextLevelElement(Optional<PsiField> current, int maxLevelIndex) {
        int index = 1;
        while (current.isPresent() && index <= maxLevelIndex) {
            String nextLevelIndexFiledName = fieldNameWithReferenceList.get(index);
            if (nextLevelIndexFiledName.contains(" ")) {
                return Optional.empty();
            }
            current = resolveReferenceField(current.get(), nextLevelIndexFiledName);
            index++;
        }
        return current;
    }

    /**
     * 从引用类型中解析字段
     * 例如user.name
     * 那么current为前面解析出来的user引用，fieldName为name
     *
     * @param current   当前引用
     * @param fieldName 要从引用中解析的字段名称
     * @return 字段
     */
    @NotNull
    private Optional<PsiField> resolveReferenceField(@NotNull PsiField current, @NotNull String fieldName) {
        PsiType type = current.getType();
        // 引用类型，而且不包含可变参数
        if (type instanceof PsiClassReferenceType && !((PsiClassReferenceType) type).hasParameters()) {
            PsiClass clazz = ((PsiClassReferenceType) type).resolve();
            if (null != clazz) {
                return JavaUtils.findSettablePsiField(clazz, fieldName);
            }
        }
        return Optional.empty();
    }
}
```
外部调用`resolve`方法，首先对一级属性进行解析，如果包含了嵌套引用，那么需要逐层解析。

有了解析器，就可以创建一个`PsiReference`引用，顾名思义，就是创建一个Psi元素的引用：
```java
public class ContextPsiFieldReference extends PsiReferenceBase<XmlAttributeValue> {

    private PsiFiledReferenceSetResolver resolver;

    /**
     * 当前解析层级
     * 例如property为"user.name"
     * 那么如果鼠标点击的user，index为1
     * 如果鼠标点击的是name，index为2
     */
    private int index;

    /**
     * @param element 当前元素
     * @param range   元素边界
     * @param index   引用层级
     */
    ContextPsiFieldReference(XmlAttributeValue element, TextRange range, int index) {
        super(element, range, false);
        this.index = index;
        resolver = new PsiFiledReferenceSetResolver(element);
    }

    /**
     * 解析xml属性，返回对应的引用变量
     */
    @SuppressWarnings("unchecked")
    @Nullable
    @Override
    public PsiElement resolve() {
        Optional<PsiElement> resolved = (Optional<PsiElement>) resolver.resolve(index);
        return resolved.orElse(null);
    }

    /**
     * 代码自动提示类型中所有可赋值（非static和非final）的成员变量列表
     *
     * @return 类型中所有可赋值（非static和非final）的成员变量列表
     */
    @NotNull
    @Override
    public Object[] getVariants() {
        Optional<PsiClass> clazz = getTargetClazz();
        return clazz.isPresent() ? JavaUtils.findSettablePsiFields(clazz.get()) : PsiReference.EMPTY_ARRAY;
    }

    /**
     * 获取property参数的类型
     * 如果是简单的"name"，那么就使用外层标签指定的类型
     * 如果是带引用的"user.name"，那么需要深入解析出具体引用类型user
     *
     * @return property参数的类型
     */
    @SuppressWarnings("unchecked")
    private Optional<PsiClass> getTargetClazz() {
        String elementValue = getElement().getValue();
        if (elementValue == null) {
            return Optional.empty();
        }
        if (elementValue.contains(String.valueOf(ReferenceSetBase.DOT_SEPARATOR))) {
            // 包含点号，说明此参数的类型为引用类型
            // 例如一个类的成员变量中包含其他引用类型
            int ind = 0 == index ? 0 : index - 1;
            Optional<PsiElement> resolved = (Optional<PsiElement>) resolver.resolve(ind);
            if (resolved.isPresent()) {
                return JavaService.getInstance(myElement.getProject()).getReferenceClazzOfPsiField(resolved.get());
            }
        } else { // 没有点号，说明参数类型不是引用类型，直接就是外层标签定义的类型（如resultMap中type指定的类型）中包含的简单类型，如基本类型或者字符串类型变量
            return MapperUtils.getPropertyClazz(myElement);
        }
        return Optional.empty();
    }

}
```
上面创建的`ContextPsiFieldReference`引用需要传入`index`变量，用来指出引用的层级。
例如xml中属性值为`user.name`，那么如果是`user`这一层的引用，`index`为1，如果是`name`这一层的引用，`index`为2。

接下来创建一个`ReferenceSetBase`，用来解析嵌套引用。
继承`ReferenceSetBase`并重写`createReference`方法，使用我们上面创建的`ContextPsiFieldReference`。基类`ReferenceSetBase`中包含着对嵌套引用的解析操作：
```java
public class ResultPropertyReferenceSet extends ReferenceSetBase<PsiReference> {

    public ResultPropertyReferenceSet(String text, @NotNull PsiElement element, int offset) {
        super(text, element, offset, ReferenceSetBase.DOT_SEPARATOR);
    }

    @Override
    protected PsiReference createReference(TextRange range, int index) {
        XmlAttributeValue element = (XmlAttributeValue) getElement();
        return null == element ? null : new ContextPsiFieldReference(element, range, index);
    }
}
```
有了引用，接下来需要放到转换器中，与字段绑定：
```java
public class PropertyConverter extends ResolvingConverter<XmlAttributeValue> implements CustomReferenceConverter<XmlAttributeValue> {

    /**
     * converter实现CustomReferenceConverter，这样能通过实现createReferences创建引用关系
     * @param value
     * @param element
     * @param context
     * @return
     */
    @NotNull
    @Override
    public PsiReference[] createReferences(GenericDomValue<XmlAttributeValue> value, PsiElement element, ConvertContext context) {
        String stringValue = value.getStringValue();
        if (stringValue == null) {
            return PsiReference.EMPTY_ARRAY;
        }
        return new ResultPropertyReferenceSet(stringValue, element, ElementManipulators.getOffsetInElement(element)).getPsiReferences();
    }

    @NotNull
    @Override
    public Collection<? extends XmlAttributeValue> getVariants(ConvertContext context) {
        return Collections.emptyList();
    }

    @Nullable
    @Override
    public XmlAttributeValue fromString(@Nullable String s, ConvertContext context) {
        DomElement ctxElement = context.getInvocationElement();
        return ctxElement instanceof GenericAttributeValue ? ((GenericAttributeValue) ctxElement).getXmlAttributeValue() : null;
    }

    @Nullable
    @Override
    public String toString(@Nullable XmlAttributeValue attributeValue, ConvertContext context) {
        return null;
    }

}
```
最后再看一次使用方式：
```java
/**
 * Description:
 * 包含property属性的元素
 *
 * @author damon4u
 * @version 2018-10-18 17:52
 */
public interface PropertyAttributeDomElement extends DomElement {

    @Attribute("property")
    @Convert(PropertyConverter.class)
    GenericAttributeValue<XmlAttributeValue> getProperty();
}
```
这样，再DOM解析时，会使用该converter，而converter实现了`CustomReferenceConverter`，可以创建引用，就能实现引用绑定了。

参考：
[PSI References](https://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_references.html)
[References and Resolve](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/custom_language_support/references_and_resolve.html)
[Reference Contributor](https://www.jetbrains.org/intellij/sdk/docs/tutorials/custom_language_support/reference_contributor.html)
