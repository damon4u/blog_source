---
title: IDEA插件开发（二）DOM操作XML文件
date: 2018-10-10 16:41:50
tags: idea
categories: [idea,java]
---


### 传统XML读写操作的弊端
假如我们有一个xml文件如下：
```xml
<root>
    <foo>
        <bar>42</bar>
        <bar>239</bar>
    </foo>
</root>
```
我们想读取第二个bar元素的值239，我们可能会直接这样写：
```java
file.getDocument().getRootTag().findFirstSubTag("foo").
findSubTags("bar")[1].getValue().getTrimmedText()
```
但这样是很危险的，因为任何一个元素都可能产生空指针。所以需要这样写：
```java
XmlFile file = ...;
final XmlDocument document = file.getDocument();
if (document != null) {
    final XmlTag rootTag = document.getRootTag();
    if (rootTag != null) {
        final XmlTag foo = rootTag.findFirstSubTag("foo");
        if (foo != null) {
            final XmlTag[] bars = foo.findSubTags("bar");
            if (bars.length > 1) {
                String s = bars[1].getValue().getTrimmedText();
                // do something
            }
        }
    }
}
```
这样的写法看起来很臃肿，有一种更好的方案是使用DOM。

### 使用DOM操作XML
用 __Document Object Model__ (DOM) 来改写上面的写法。
首先需要继承`DomElement`，为每一个xml元素创建接口：
```java
interface Root extends com.intellij.util.xml.DomElement {
    Foo getFoo();
}

interface Foo extends com.intellij.util.xml.DomElement {
    List<Bar> getBars();
}

interface Bar extends com.intellij.util.xml.DomElement {
    String getValue();
}
```

接下来需要继承`DomFileDescription`，构造函数中传入根元素名字和根元素接口，并注册到扩展点`com.intellij.dom.fileDescription`中。
```java
public class MyDescription extends DomFileDescription<Root> {

    public MyDescription() {
        super(Root.class, "root");
    }
}
```
注册：
```xml
<extensions defaultExtensionNs="com.intellij">
    <dom.fileDescription implementation="MyDescription"/>
</extensions>
```
接下来就可以只用`DomManager`获取元素了：
```java
DomManager manager = DomManager.getDomManager(project);
Root root = manager.getFileElement(file).getRootElement();
List<Bar> bars = root.getFoo().getBars();
if (bars.size() > 1) {
    String s = bars.get(1).getValue();
    // do something
}
```
有时候我们想要从项目的xml文件中找到某个`DomElement`进行操作，可以使用`DomService`：
```java
static <T extends DomElement> Collection<T> findDomElements(@NotNull Project project, Class<T> clazz) {
    GlobalSearchScope scope = GlobalSearchScope.allScope(project);
    List<DomFileElement<T>> elements = DomService.getInstance().getFileElements(clazz, project, scope);
    return Collections2.transform(elements, DomFileElement::getRootElement);
}
```

参考资料
[官方文档XML DOM API](https://www.jetbrains.org/intellij/sdk/docs/reference_guide/frameworks_and_external_apis/xml_dom_api.html)
