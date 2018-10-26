---
title: IDEA插件开发（十一）mybatis插件之Xml校验
date: 2018-10-26 18:41:50
tags: idea
categories: [idea,java]
---

今天要实现的功能是，检查xml中的增删改查有没有对应的DAO层方法，如果没有，则报错。

IDEA插件中是通过 __Inspection__ 来实现这个功能。

对XML的检查，IDEA提供了一个`BasicDomElementsInspection`来做基本检查，我们需要做的就是做好DOM解析工作，对关键属性添加`converter`。最后注册到`plugin.xml`即可。

<!-- more -->

为了让`BasicDomElementsInspection`帮我们做Dom元素检查，我们首先需要为属性添加约束条件。
例如，我们想让增删改查对应这DAO层方法，那我们就可以为他们的id属性添加一个`ResolvingConverter`:

先添加一个适配器，提供默认实现方法，具体减少实现类的无用代码：
```java
public abstract class ConverterAdaptor<T> extends ResolvingConverter<T> {

    @NotNull
    @Override
    public Collection<? extends T> getVariants(ConvertContext context) {
        return Collections.emptyList();
    }

    @Nullable
    @Override
    public T fromString(@Nullable String s, ConvertContext context) {
        return null;
    }

    @Nullable
    @Override
    public String toString(@Nullable T t, ConvertContext context) {
        return null;
    }
}
```
然后就创建一个DAO层方法绑定的实现，重写`fromString()`方法，根据命名空间和id（即方法名称）查询对应的DAO方法：
```java
public class DaoMethodConverter extends ConverterAdaptor<PsiMethod> {

    @Nullable
    @Override
    public PsiMethod fromString(@Nullable @NonNls String id, ConvertContext context) {
        Mapper mapper = MapperUtils.getMapper(context.getInvocationElement());
        return JavaUtils.findMethod(context.getProject(), MapperUtils.getNamespace(mapper), id).orElse(null);
    }
}
```
然后绑定到增删改查都实现的接口上：
```java
public interface DMLAndDQLDynamicQueryableDomElement extends DynamicQueryableDomElement, IdDomElement {

    /**
     * 重写id属性，提供转换器，增删改查会实现这个接口，用来对应Dao层方法
     */
    @Attribute("id")
    @Convert(DaoMethodConverter.class)
    GenericAttributeValue<String> getId();

    @NotNull
    @Attribute("parameterType")
    // TODO: converter
    GenericAttributeValue<PsiClass> getParameterType();
}
```

最后，我们实现一个`BasicDomElementsInspection`即可：
```java
/**
 * Description:
 *
 * 检查mapper文件中的语句有没有对应的dao层方法
 * 继承自BasicDomElementsInspection，依赖于DOM解析过程中的规则，
 * 需要为ParameteredDynamicQueryableDomElement中的id添加DaoMethodConverter，绑定对应关系
 *
 * @author damon4u
 * @version 2018-10-25 21:49
 */
public class MapperXmlInspection extends BasicDomElementsInspection<DomElement> {

    public MapperXmlInspection() {
        super(DomElement.class);
    }

    @Override
    protected void checkDomElement(DomElement element, DomElementAnnotationHolder holder, DomHighlightingHelper helper) {
        super.checkDomElement(element, holder, helper);
    }

}
```
注册到`plugin.xml`中：
```xml
<localInspection language="XML" shortName="MybatisMapperXmlInspection" enabledByDefault="true" level="ERROR"
                 displayName="Mapper xml inspection"
                 groupName="Mybatis"
                 implementationClass="com.damon4u.plugin.mybatis.inspection.MapperXmlInspection"/>
```

还有一点小完善，就是为这个inspection创建说明文档，这个文档在settings对话框中展示。如果没有提供这样的说明文档，IDEA会给提示。
在`/resources/inspectionDescriptions`下创建`MybatisMapperXmlInspection.html`：
```html
<html>
<body>
<p>There should be a method in DAO interface with the same name of Mybatis XML tag.</p>
<p>Add a method according to the XML tag id or just remove the tag.</p>
</body>
</html>
```
实现的效果：
检测到没有DAO层方法：

![](/images/idea-plugin18.png)

打开setting提示：
![](/images/idea-plugin19.png)

参考：
[Code Inspections](https://www.jetbrains.org/intellij/sdk/docs/tutorials/code_inspections.html)
[Code Inspection](https://www.jetbrains.com/help/idea/code-inspection.html)
