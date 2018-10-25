---
title: IDEA插件开发（九）mybatis插件之参数补全
date: 2018-10-25 17:41:50
tags: idea
categories: [idea,java]
---

IDEA插件开发时，可以有两种途径提供代码补全：
* 实现`Reference`的`getVariants()`方法，返回一个数组，类型为`String`或者`PsiElement`或者`LookupElement`。这种方式只支持基本（basic）补全操作。
* 继承`CompletionContributor`，支持basic、smart和class name三种补全方式。

本文主要介绍继承`CompletionContributor`的方式。

<!-- more -->

先说目标。

还是以之前的mapper文件为例：
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

我想为sql语句和if条件中的`#{userName}`添加自动补全。

首先创建一个`CompletionContributor`的基本实现类，其中包含公共方法，为元素添加自动补全：
```java
abstract class BaseParamCompletionContributor extends CompletionContributor {

    private static final double PRIORITY = 400.0;

    static void addCompletionForPsiParameter(@NotNull final Project project,
                                             @NotNull final CompletionResultSet completionResultSet,
                                             @Nullable final IdDomElement element) {
        if (element == null) {
            return;
        }

        final PsiMethod method = JavaUtils.findMethod(project, element).orElse(null);

        if (method == null) {
            return;
        }

        final PsiParameter[] parameters = method.getParameterList().getParameters();

        if (parameters.length == 1) {
            final PsiParameter parameter = parameters[0];
            completionResultSet.addElement(buildLookupElementWithIcon(parameter.getName(), parameter.getType().getPresentableText()));
        } else {
            for (PsiParameter parameter : parameters) {
                final Optional<String> annotationValueText = JavaUtils.getAnnotationValueText(parameter, Annotation.PARAM);
                completionResultSet.addElement(buildLookupElementWithIcon(annotationValueText.orElseGet(parameter::getName), parameter.getType().getPresentableText()));
            }
        }

    }

    private static LookupElement buildLookupElementWithIcon(final String parameterName,
                                                            final String parameterType) {
        return PrioritizedLookupElement.withPriority(
                LookupElementBuilder.create(parameterName)
                        .withTypeText(parameterType)
                        .withIcon(PlatformIcons.PARAMETER_ICON),
                PRIORITY);
    }

}
```
该方法的逻辑比较清楚，先查出元素所在方法的参数列表，然后将参数构造成`LookupElement`返回。
构造`LookupElement`时，需要设置参数名称，参数类型和图标。
实现方式与上一篇[IDEA插件开发（八）mybatis插件之参数引用](https://damon4u.github.io/blog/2018/10/IDEA%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91%EF%BC%88%E5%85%AB%EF%BC%89mybatis%E6%8F%92%E4%BB%B6%E4%B9%8B%E5%8F%82%E6%95%B0%E5%BC%95%E7%94%A8.html) 类似，不多解释。

主要看两个具体的实现类。

#### 重写fillCompletionVariants方法

第一个，为sql语句中的`#{userName}`添加自动补全。
```java
public class SqlParamCompletionContributor extends BaseParamCompletionContributor {


    @Override
    public void fillCompletionVariants(@NotNull CompletionParameters parameters, @NotNull CompletionResultSet result) {
        if (parameters.getCompletionType() != CompletionType.BASIC) {
            return;
        }
        final PsiElement position = parameters.getPosition();
        PsiFile topLevelFile = InjectedLanguageManager.getInstance(parameters.getPosition().getProject()).getTopLevelFile(position);
        if (DomUtils.isMybatisFile(topLevelFile)) {
            if (shouldAddElement(position.getContainingFile(), parameters.getOffset())) {
                process(topLevelFile, result, position);
            }    
        }

    }

    private boolean shouldAddElement(PsiFile file, int offset) {
        String text = file.getText();
        for (int i = offset - 1; i > 0; i--) {
            char c = text.charAt(i);
            if (c == '{' && text.charAt(i - 1) == '#') {
                return true;
            }
        }
        return false;
    }

    private void process(PsiFile xmlFile, CompletionResultSet result, PsiElement position) {
        int offset = InjectedLanguageManager.getInstance(position.getProject()).injectedToHost(position, position.getTextOffset());
        Optional<IdDomElement> idDomElement = MapperUtils.findParentIdDomElement(xmlFile.findElementAt(offset));
        if (idDomElement.isPresent()) {
            addCompletionForPsiParameter(position.getProject(), result, idDomElement.get());
            result.stopHere();
        }
    }
}
```
为了方便解释，我们先把注册到`plugin.xml`的代码贴出来：
```xml
<completion.contributor language="SQL"
                                implementationClass="com.damon4u.plugin.mybatis.completion.SqlParamCompletionContributor"
                                order="first"/>
```
第一种方式重写`fillCompletionVariants`方法，首先也是过滤参数。
在注册时，我们指定了language为SQL，那么该补全器只会传入SQL语句中参数。
然后我们在方法内过滤mybatis的mapper文件中的SQL语句参数。
然后判断是否是`#{`开头，如果是，才补全。

#### 使用extend方法
```java
public class TestParamCompletionContributor extends BaseParamCompletionContributor {

    public TestParamCompletionContributor() {
        extend(CompletionType.BASIC,
                XmlPatterns.psiElement().inside(XmlPatterns.xmlAttribute().withName("test")),
                new CompletionProvider<CompletionParameters>() {
                    @Override
                    protected void addCompletions(@NotNull CompletionParameters parameters,
                                                  ProcessingContext context,
                                                  @NotNull CompletionResultSet result) {
                        final PsiElement position = parameters.getPosition();
                        addCompletionForPsiParameter(position.getProject(), result, MapperUtils.findParentIdDomElement(position).orElse(null));
                    }
                });
    }

}
```
常用的实现方式是在构造函数中调用`extend()`方法。
第一个参数是补全类型，支持basic，smart和class name。一般选basic就好了。
第二个参数是参数匹配模式。本例子中是找到`name`为`test`的`XmlAttribute`内部的元素：`XmlPatterns.psiElement().inside(XmlPatterns.xmlAttribute().withName("test"))`。
第三个参数是`CompletionProvider`实例，重写`addCompletions`方法即可。

最后注册到`plugin.xml`中：
```xml
<completion.contributor language="XML"
                                implementationClass="com.damon4u.plugin.mybatis.completion.TestParamCompletionContributor"/>
```

参考：
[Code Completion](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/custom_language_support/code_completion.html)
[Completion Contributor](https://www.jetbrains.org/intellij/sdk/docs/tutorials/custom_language_support/completion_contributor.html)
