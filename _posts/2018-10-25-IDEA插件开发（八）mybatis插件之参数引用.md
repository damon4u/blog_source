---
title: IDEA插件开发（八）mybatis插件之参数引用
date: 2018-10-25 17:41:50
tags: idea
categories: [idea,java]
---

[IDEA插件开发（七）mybatis插件之mapper解析](https://damon4u.github.io/blog/2018/10/IDEA%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91%EF%BC%88%E4%B8%83%EF%BC%89mybatis%E6%8F%92%E4%BB%B6%E4%B9%8Bmapper%E8%A7%A3%E6%9E%90.html) 一文中使用到了`PropertyConverter`来为`property`属性添加引用，实现鼠标点击跳转，代码补全提示等功能，其中使用了`ContextPsiFieldReference`，它继承了`PsiReferenceBase`，实现`resolve()`方法来返回对应的引用对象。

今天主要就说说引用（ __PSI References__ ），以及如何单独注册引用。

> A reference in a PSI tree is an object that represents a link from a usage of a certain element in the code to the corresponding declaration. Resolving a reference means locating the declaration to which a specific usage refers.

> Resolving references gives users the ability to navigate from a PSI element usage (accessing a variable, calling a method and so on) to the declaration of that element (the variable’s definition, a method declaration and so on). This feature is needed in order to support the `Go to Declaration` action invoked by __Ctrl-B__ and __Ctrl-Click__, and it is a prerequisite for implementing the Find Usages action, the Rename refactoring and code completion.

举个例子，也就是我想要实现的效果。
Dao层有个方法：
```java
List<User> findByUserName(@Param("userName") String userName);
```
然后对应的mapper文件中的查询语句：
```xml
<select id="findByUserName" resultType="com.damon4u.demo.domain.User">
    select id, user_name
    from user
    <where>
        <if test="userName != null">
            user_name like CONCAT('%',#{userName},'%')
        </if>
    </where>
</select>
```
我现在想实现点击`#{userName}`中的`userName`能够跳转到Dao层方法的`userName`参数，即为xml中的参数添加引用。

<!-- more -->

要实现这样的功能，有两个关键点：
* 创建一个`PsiReferenceBase`引用类，实现`resolve()`方法，返回值为当前`PsiElement`需要跳转到的目标`PsiElement`。
* 继承`PsiReferenceContributor`，实现`registerReferenceProviders()`方法，将创建的引用类注册（绑定）到一个触发`PsiElement`上。可以理解为点击哪个元素会触发引用跳转。

### 引用绑定注册器PsiReferenceContributor
按照逻辑，我们需要先找到触发元素，本例中，就是先要从mapper文件中找出`#{userName}`中的`userName`，即符合`#{paramName}`模式的参数名。然后将引用绑定到该元素上。
寻找和绑定的逻辑需要放到`PsiReferenceContributor`中：
```java
public class SqlParamReferenceContributor extends PsiReferenceContributor {

    // 匹配 #{xxx}
    private static final Pattern PARAM_PATTERN = Pattern.compile("#\\{(.*?)}");

    @Override
    public void registerReferenceProviders(@NotNull PsiReferenceRegistrar registrar) {
        registrar.registerReferenceProvider(PlatformPatterns.psiElement(XmlToken.class),
                new PsiReferenceProvider() {
                    @NotNull
                    @Override
                    public PsiReference[] getReferencesByElement(@NotNull PsiElement element, @NotNull ProcessingContext context) {
                        XmlToken token = (XmlToken) element;
                        // XmlToken所在文件需要是mapper
                        if (MapperUtils.isElementWithinMybatisFile(token)) {
                            String text = token.getText();
                            Matcher matcher = PARAM_PATTERN.matcher(text);
                            ArrayList<PsiReference> referenceList = Lists.newArrayList();
                            // 正则匹配出#{paramName}中的paramName，并为每一个参数添加引用
                            while (matcher.find()) {
                                // 参数名
                                String param = matcher.group(1);
                                // 在XmlToken内容中的开始位置
                                int start = matcher.start(1);
                                // 在XmlToken内容中的结束位置
                                int end = matcher.end(1);
                                referenceList.add(new ParamReference(token, new TextRange(start, end), param));
                            }
                            return referenceList.toArray(new PsiReference[0]);
                        }

                        return PsiReference.EMPTY_ARRAY;
                    }
                });
    }
}
```
重点在于`registerReferenceProvider`的两个参数。
第一个参数传入`PlatformPatterns.psiElement(XmlToken.class)`，代表我们要为`XmlToken`添加引用。
为什么说我们要为`XmlToken`添加引用呢？因为mapper的xml文件中，最基本的元素就是一个`XmlToken`，`user_name`是一个，`like`是一个，`CONCAT('%',#{userName},'%')`是一个。
可以简单的理解为，使用空格分割的元素都是。
为了分析IDEA中的元素从属于哪种`PsiElement`类型，可以给沙箱安装一个插件`PsiViewer`，然后就可以方便的分析`PsiElement`类型了：

![](/images/idea-plugin17.png)

鼠标点击要分析的元素上，右边能清晰的看到元素的属性以及层级关系。

第二个参数是`PsiReferenceProvider`实现类，主要实现`getReferencesByElement`方法，将引用绑定到指定元素上。
由于所有xml中的`XmlToken`都会被放到候选集中，我们需要过滤出想要的元素，必须在mapper文件内。
```java
public static boolean isElementWithinMybatisFile(@NotNull PsiElement element) {
    PsiFile psiFile = element.getContainingFile();
    return element instanceof XmlElement && DomUtils.isMybatisFile(psiFile);
}
```
`DomUtils.isMybatisFile(psiFile);`方法比较简单，就是比较根标签是否是`mapper`。

之后，使用正则表达式取出符合`#{paramName}`模式的参数名，以及参数名在`XmlToken`中的开始位置和结束位置，构造一个TextRange，作为鼠标点击区域。
之后构造一个我们自己的引用实例`ParamReference`，加入到返回列表中，完成绑定。

### 引用PsiReferenceBase
接下来就要下我们自己的`PsiReferenceBase`了：
```java
public class ParamReference extends PsiReferenceBase<XmlElement> {

    // 参数名称
    private String paramName;

    public ParamReference(@NotNull XmlElement attributeValue, TextRange textRange, @NotNull String paramName) {
        // 调用父类构造函数
        super(attributeValue, textRange);
        this.paramName = paramName;
    }

    @Nullable
    @Override
    public PsiElement resolve() {
        XmlElement element = getElement();
        Project project = element.getProject();
        // 寻找当前元素的父级IdDomElement，这里应该拿到的是Select，Update等，包含Dao层方法名
        IdDomElement domElement = MapperUtils.findParentIdDomElement(element).orElse(null);
        if (domElement == null) {
            return null;
        }
        // 根据mapper的namespace拿到Dao类，然后根据IdDomElement的id拿到方法名称
        final PsiMethod method = JavaUtils.findMethod(project, domElement).orElse(null);
        if (method == null) {
            return null;
        }
        // 取出方法参数
        final PsiParameter[] parameters = method.getParameterList().getParameters();

        // dao层方法参数可能有两种情况
        // 1、只有一个参数，那么可能这个参数没有用@Param注解标注，那么直接使用参数名称与paramName比较；
        // 2、方法有多个参数，那么每个参数都应该使用@Param注解标注，用@Param的value值挨个与paramName比较。如果没有标注，那么就认为没找到，不创建引用。
        if (parameters.length == 1) {
            PsiParameter parameter = parameters[0];
            if (paramName.equals(parameter.getName())) {
                return parameter;
            } else {
                return null;
            }
        } else {
            for (final PsiParameter parameter : parameters) {
                final Optional<String> value = JavaUtils.getAnnotationValueText(parameter, Annotation.PARAM);
                if (value.isPresent() && paramName.equals(value.get())) {
                    return parameter;
                }
            }
        }
        return null;
    }

    /**
     * 这个方法是用来提供代码补全候选的，这里没有实现，后面会用SqlParamCompletionContributor实现补全
     */
    @NotNull
    @Override
    public Object[] getVariants() {
        return new Object[0];
    }
}
```
其中设计到`JavaUtils`中的几个方法，寻找mapper对应的dao层方法：
```java
@NotNull
public static Optional<PsiMethod> findMethod(@NotNull Project project, @NotNull IdDomElement element) {
    return findMethod(project, MapperUtils.getNamespace(element), MapperUtils.getId(element));
}

@NotNull
public static Optional<PsiMethod> findMethod(@NotNull Project project, @Nullable String clazzName, @Nullable String methodName) {
    if (StringUtils.isBlank(clazzName) || StringUtils.isBlank(methodName)) {
        return Optional.empty();
    }
    Optional<PsiClass> clazz = findClazz(project, clazzName);
    if (clazz.isPresent()) {
        PsiMethod[] methods = clazz.get().findMethodsByName(methodName, true);
        return ArrayUtils.isEmpty(methods) ? Optional.empty() : Optional.of(methods[0]);
    }
    return Optional.empty();
}

@NotNull
public static Optional<PsiClass> findClazz(@NotNull Project project, @NotNull String clazzName) {
    return Optional.ofNullable(JavaPsiFacade.getInstance(project).findClass(clazzName, GlobalSearchScope.allScope(project)));
}
```
最后，将`SqlParamReferenceContributor`注册到`plugin.xml`中：
```xml
<psi.referenceContributor implementation="com.damon4u.plugin.mybatis.reference.SqlParamReferenceContributor"/>
```

至此，便完成了对sql语句中`#{userName}`的引用创建。

下面再举一个例子，并说说开发时遇到的坑。
回到刚才的mapper文件中的查询语句：
```xml
<select id="findByUserName" resultType="com.damon4u.demo.domain.User">
    select id, user_name
    from user
    <where>
        <if test="userName != null">
            user_name like CONCAT('%',#{userName},'%')
        </if>
    </where>
</select>
```
我现在还想为`<if test="userName != null">`中的`userName`添加引用，同样跳转到Dao层方法参数。

#### 思路一，完全复用SqlParamReferenceContributor
使用PsiViewer分析`<if test="userName != null">`也属于`XmlToken`类型，那理论上`SqlParamReferenceContributor`也能接收到。
但是反复Debug都没有拦截到这个值。后来发现，只有`XmlText`下的`XmlToken`才能被拦截到。
那可能是`PlatformPatterns.psiElement(XmlToken.class)`这种创建Patterns的方式有问题，只能拦截`XmlText`下的`XmlToken`，不能拦截`XmlAttributeValue`下的`XmlToken`。
所以这个思路不行。

#### 思路二，匹配XmlAttribute
只要能取出所有mapper中的`XmlAttribute`，然后比较`name`是否为`test`，如果是，取出`XmlAttributeValue`，匹配关键字创建引用即可。
开发过程中，发现匹配没问题，但是会把引用绑定到`XmlAttribute`整体上，`test="userName != null"`。
其实我们真正想要的是给`XmlAttributeValue`创建引用，即`"userName != null"`部分。

#### 思路三，匹配XmlAttributeValue
最开始也是想匹配`XmlAttributeValue`的，但是不好过滤前面的`test`。所以想简单的匹配`XmlAttribute`，然后比较`name`是否为`test`。
`XmlAttributeValue`其实是`XmlAttribute`的子标签，获取父类即可拿到`XmlAttribute`。
下面看实现：
```java
public class TestParamReferenceContributor extends PsiReferenceContributor {

    // 匹配参数值，这里就去掉了"!=" "=="等运算符和数字
    private static final Pattern PARAM_PATTERN = Pattern.compile("([A-Za-z]+)");

    @Override
    public void registerReferenceProviders(@NotNull PsiReferenceRegistrar registrar) {
        // 注意，这里第一个参数说明是要给XmlAttributeValue创建引用，那么下面返回TestParamReference的泛型参数也是XmlAttributeValue
        // 开始试过一次XmlAttribute来作为pattern，然后TestParamReference的泛型参数是XmlAttributeValue，这样其实需要点击外层属性name才能触发，不对应
        registrar.registerReferenceProvider(PlatformPatterns.psiElement(XmlAttributeValue.class),
                new PsiReferenceProvider() {
                    @NotNull
                    @Override
                    public PsiReference[] getReferencesByElement(@NotNull PsiElement element, @NotNull ProcessingContext context) {
                        XmlAttributeValue xmlAttributeValue = (XmlAttributeValue) element;
                        if (MapperUtils.isElementWithinMybatisFile(xmlAttributeValue)) {
                            // XmlAttributeValue其实是XmlAttribute的子标签，获取父类即可拿到XmlAttribute
                            PsiElement xmlAttribute = xmlAttributeValue.getParent();
                            if (xmlAttribute instanceof XmlAttribute && ((XmlAttribute) xmlAttribute).getName().equals("test")) {
                                String value = xmlAttributeValue.getValue();
                                if (StringUtils.isNotBlank(value)) {
                                    ArrayList<PsiReference> referenceList = Lists.newArrayList();
                                    Matcher matcher = PARAM_PATTERN.matcher(value);
                                    while (matcher.find()) {
                                        String param = matcher.group(1);
                                        // 注意需要排除一些逻辑关键字
                                        if (param.equalsIgnoreCase("and")
                                                || param.equalsIgnoreCase("or")
                                                || param.equalsIgnoreCase("null")) {
                                            continue;
                                        }
                                        // 这里加1是因为attribute value用双引号包着
                                        int start = matcher.start(1) + 1;
                                        int end = matcher.end(1) + 1;
                                        referenceList.add(new ParamReference(xmlAttributeValue, new TextRange(start, end), param));
                                    }
                                    return referenceList.toArray(new PsiReference[0]);
                                }
                            }
                        }

                        return PsiReference.EMPTY_ARRAY;
                    }
                });
    }

}
```
其中引用可以复用之前的`ParamReference`。
最后不要忘了注册到`plugin.xml`中：
```xml
<psi.referenceContributor implementation="com.damon4u.plugin.mybatis.reference.TestParamReferenceContributor"/>
```

参考：
[PSI References](https://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_references.html)
[References and Resolve](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/custom_language_support/references_and_resolve.html)
[Reference Contributor](https://www.jetbrains.org/intellij/sdk/docs/tutorials/custom_language_support/reference_contributor.html)
