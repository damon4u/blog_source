---
title: IDEA插件开发（十）mybatis插件之DAO与XML之间跳转
date: 2018-10-26 17:41:50
tags: idea
categories: [idea,java]
---

使用Mybatis进行开发时，还有一个常用场景是从DAO层方法跳转到XML标签，或者反过来从XML标签跳转到DAO层方法。
可以使用`lineMarkerProvider`来实现该功能。

> Line markers help to annotate any code with icons on the gutter. These icons may provide navigation to related code.

<!-- more -->

### DAO层方法跳转到对应XML标签
```java
public class DaoToXmlLineMarkerProvider extends RelatedItemLineMarkerProvider {

    private static final Function<DomElement, XmlTag> FUNCTION = DomElement::getXmlTag;

    @Override
    protected void collectNavigationMarkers(@NotNull PsiElement element, @NotNull Collection<? super RelatedItemLineMarkerInfo> result) {
        if (element instanceof PsiNameIdentifierOwner && JavaUtils.isElementWithinInterface(element)) {
            CommonProcessors.CollectProcessor<IdDomElement> processor = new CommonProcessors.CollectProcessor<>();
            JavaService.getInstance(element.getProject()).process(element, processor);
            Collection<IdDomElement> results = processor.getResults();
            if (!results.isEmpty()) {
                NavigationGutterIconBuilder<PsiElement> builder = NavigationGutterIconBuilder.create(Icons.DAO_TO_XML_LINE_MARKER_ICON)
                        .setAlignment(GutterIconRenderer.Alignment.CENTER)
                        .setTargets(Collections2.transform(results, FUNCTION))
                        .setTooltipTitle("Navigation to target in mapper xml");
                PsiElement nameIdentifier = ((PsiNameIdentifierOwner) element).getNameIdentifier();
                if (nameIdentifier != null) {
                    result.add(builder.createLineMarkerInfo(nameIdentifier));
                }
            }
        }
    }
}
```
同样先过滤出DAO层接口，可能是接口本身，也可能是接口中的方法。
根据接口或者方法`PsiElement`寻找对应`IdDomElement`的逻辑在`JavaService#process()`方法中：
```java
public void process(@NotNull PsiElement target, @NotNull Processor processor) {
    if (target instanceof PsiMethod) { // dao方法
        process((PsiMethod) target, processor);
    } else if (target instanceof PsiClass) { // dao接口
        process((PsiClass) target, processor);
    }
}

public void process(@NotNull PsiMethod psiMethod, @NotNull Processor<IdDomElement> processor) {
    PsiClass psiClass = psiMethod.getContainingClass();
    if (null == psiClass) return;
    String id = psiClass.getQualifiedName() + "." + psiMethod.getName();
    for (Mapper mapper : MapperUtils.findMappers(psiMethod.getProject())) { // 先根据命名空间找mapper
        for (IdDomElement idDomElement : mapper.getDaoElements()) {
            if (MapperUtils.getIdSignature(idDomElement).equals(id)) { // 对比tag的id与方法名
                processor.process(idDomElement);
            }
        }
    }
}

public void process(@NotNull PsiClass clazz, @NotNull Processor<Mapper> processor) {
    String ns = clazz.getQualifiedName();
    for (Mapper mapper : MapperUtils.findMappers(clazz.getProject())) {
        if (MapperUtils.getNamespace(mapper).equals(ns)) {
            processor.process(mapper);
        }
    }
}
```
找到符合的`IdDomElement`后，构造一个跳转`LineMarkerInfo`：
需要传入图标、布局、跳转目标和提示语。

### XML标签跳转到DAO层方法
```java
public class XmlToDaoLineMarkerProvider implements LineMarkerProvider {

    private static final ImmutableList<Class<? extends DMLAndDQLDynamicQueryableDomElement>> TARGET_TYPES = ImmutableList.of(
            Select.class,
            Update.class,
            Insert.class,
            Delete.class
    );

    @Nullable
    @Override
    public LineMarkerInfo getLineMarkerInfo(@NotNull PsiElement element) {
        if (!isTargetElement(element)) {
            return null;
        }
        DomElement domElement = DomUtil.getDomElement(element);
        if (domElement == null) {
            return null;
        }
        Optional<PsiMethod> method = JavaUtils.findMethod(element.getProject(), (IdDomElement) domElement);
        return method.map(psiMethod -> new LineMarkerInfo<>((XmlTag) element,
                element.getTextRange(),
                Icons.XML_TO_DAO_LINE_MARKER_ICON,
                Pass.UPDATE_ALL,
                getTooltipProvider(psiMethod),
                getNavigationHandler(psiMethod),
                GutterIconRenderer.Alignment.CENTER)).orElse(null);
    }

    @Override
    public void collectSlowLineMarkers(@NotNull List<PsiElement> elements, @NotNull Collection<LineMarkerInfo> result) {

    }

    private boolean isTargetElement(@NotNull PsiElement element) {
        return element instanceof XmlTag
                && MapperUtils.isElementWithinMybatisFile(element)
                && isTargetType(element);
    }

    private boolean isTargetType(PsiElement element) {
        DomElement domElement = DomUtil.getDomElement(element);
        for (Class<?> clazz : TARGET_TYPES) {
            if (clazz.isInstance(domElement))
                return true;
        }
        return false;
    }

    private Function<XmlTag, String> getTooltipProvider(final PsiMethod target) {
        return from -> getTooltip(target);
    }

    private GutterIconNavigationHandler<XmlTag> getNavigationHandler(final PsiMethod target) {
        return (e, from) -> ((Navigatable) target.getNavigationElement()).navigate(true);
    }

    private String getTooltip(@NotNull PsiMethod target) {
        PsiClass containingClass = target.getContainingClass();
        if (containingClass == null) {
            return "Data access object not found";
        }
        return "Data access object found - " + containingClass.getQualifiedName();
    }


}
```
这个类继承自最基本的`LineMarkerProvider`，实现`getLineMarkerInfo`方法。
同样是先过滤，然后构造`LineMarkerInfo`。

最后注册到`plugin.xml`中：
```
<!-- 在java类或者方法行添加跳转到mapper的icon -->
<codeInsight.lineMarkerProvider language="JAVA"
                                implementationClass="com.damon4u.plugin.mybatis.linemarker.DaoToXmlLineMarkerProvider"/>

<!-- 在mapper文件中添加跳转到java类的icon -->
<codeInsight.lineMarkerProvider language="XML"
                                implementationClass="com.damon4u.plugin.mybatis.linemarker.XmlToDaoLineMarkerProvider"/>
```

参考：
[Line Marker Provider](https://www.jetbrains.org/intellij/sdk/docs/tutorials/custom_language_support/line_marker_provider.html)
