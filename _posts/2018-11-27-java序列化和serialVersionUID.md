---
title: java序列化和serialVersionUID
date: 2018-11-27 16:41:50
tags: java
categories: java
---

关于jdk的序列化，可以参考`java.io.Serializable`中的javadoc，这里摘取翻译一部分：

类通过实现 java.io.Serializable 接口以启用其序列化功能。
未实现此接口的类将无法使其任何状态序列化或反序列化。
可序列化类的所有子类型本身都是可序列化的。因为实现接口也是间接的等同于继承。

<!-- more -->

关于`serialVersionUID`的描述
序列化运行时使用一个称为`serialVersionUID`的版本号与每个可序列化类相关联，该序列号在反序列化过程中用于验证序列化对象的发送者和接收者是否为该对象加载了与序列化兼容的类。
如果接收者加载的该对象的类的`serialVersionUID`与对应的发送者的类的版本号不同，则反序列化将会导致`InvalidClassException`。可序列化类可以通过声明名为`serialVersionUID`的字段（该字段必须是静态 (static)、最终 (final) 的 long 型字段）显式声明其自己的`serialVersionUID`：

如果可序列化类未显式声明`serialVersionUID`，则序列化运行时将基于该类的各个方面计算该类的默认`serialVersionUID`值，如“Java(TM) 对象序列化规范”中所述。不过，强烈建议 __所有可序列化类都显式声明 `serialVersionUID`值__，原因是计算默认的`serialVersionUID`对类的详细信息具有较高的敏感性，根据编译器实现的不同可能千差万别，这样在反序列化过程中可能会导致意外的`InvalidClassException`。因此，为保证 `serialVersionUID`值跨不同java编译器实现的一致性，序列化类必须声明一个明确的`serialVersionUID`值。还强烈建议使用`private`修饰符显示声明`serialVersionUID`（如果可能），原因是 __这种声明仅应用于直接声明类 -- `serialVersionUID` 字段作为继承成员没有用处__ 。数组类不能声明一个明确的`serialVersionUID`，因此它们总是具有默认的计算值，但是数组类没有匹配`serialVersionUID`值的要求。

之前有一种蠢写法，先创建一个`BaseEntity`：
```java
public abstract class BaseEntity implements Serializable {

    protected static final long serialVersionUID = 1L;

}
```
然后认为子类也会包含`serialVersionUID`，但是javadoc中说的很清楚，`serialVersionUID`在继承的时候没有用。
所以这种写法是错误的。

参考：
[What is a serialVersionUID and why should I use it?](https://stackoverflow.com/questions/285793/what-is-a-serialversionuid-and-why-should-i-use-it)
[Java 之 Serializable 序列化和反序列化的概念,作用的通俗易懂的解释](https://blog.csdn.net/qq_27093465/article/details/78544505)
