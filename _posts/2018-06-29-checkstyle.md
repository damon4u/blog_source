---
title: 使用checkstyle进行代码检查
date: 2018-06-29 17:41:50
tags: checkstyle
categories: checkstyle
---

> Leave the campground cleaner than you found it. (要让离开时的营地比进入时更加干净。)

作为一名程序员，对代码格式要有一些基本的要求，比如命名规范，空格等。
良好的代码规范不仅能让项目看起来干净，还能够减少低级错误，提高代码质量。
目前普遍认同阿里巴巴提供的代码规范[阿里巴巴Java开发手册](https://102.alibaba.com/downloadFile.do?file=1528269849853/Java_manual.pdf)

在团队中口头要求可能达不到约束的作用，checkstyle配合IDEA插件可以在编译器做静态代码检查。

<!-- more -->

### 1. 配置
首先安装IDEA插件`CheckStyle-IDEA`:
![](/images/checkstyle1.jpeg)
重启IDEA后，底部工具拦会出现CheckStyle窗口，点开可以看到相关功能：

![](/images/checkstyle2.jpeg)

1. 切换检查模版文件
2. 对当前文件进行检查
3. 对当前module进行检查
4. 对当前项目进行检查

其中插件自带两个模板文件，一个是Sun公司提供的，一个是Google提供的，这两个检查相对严格，不建议直接使用。
这里我们自己添加一个模版文件，只做必要的检查：
```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Puppy Crawl//DTD Check Configuration 1.3//EN"
    "http://www.puppycrawl.com/dtds/configuration_1_3.dtd">

<module name="Checker">

    <!-- 检查文件是否以一个空行结束 -->
    <module name="NewlineAtEndOfFile"/>

    <!-- 文件长度不超过1500行 -->
    <module name="FileLength">
        <property name="max" value="1500"/>
     </module>

    <!-- 每个java文件一个语法树 -->
    <module name="TreeWalker">
        <!-- import检查-->
        <!-- 避免使用* -->
        <module name="AvoidStarImport">
            <property name="excludes" value="java.io,java.net,java.lang.Math"/>
            <!-- 实例；import java.util.*;.-->
            <property name="allowClassImports" value="false"/>
            <!-- 实例 ；import static org.junit.Assert.*;-->
            <property name="allowStaticMemberImports" value="true"/>
        </module>
        <!-- 检查是否从非法的包中导入了类 -->
        <module name="IllegalImport"/>
        <!-- 检查是否导入了多余的包 -->
        <module name="RedundantImport"/>
        <!-- 没用的import检查，比如：1.没有被用到2.重复的3.import java.lang的4.import 与该类在同一个package的 -->
        <module name="UnusedImports" />

        <!-- 注释检查 -->
        <!-- 检查构造函数的javadoc -->
        <module name="JavadocType">
            <property name="allowUnknownTags" value="true"/>
            <message key="javadoc.missing" value="类注释：缺少Javadoc注释。"/>
        </module>

        <!-- 命名检查 -->
        <!-- 局部的final变量，包括catch中的参数的检查 -->
        <module name="LocalFinalVariableName" />
        <!-- 局部的非final型的变量，包括catch中的参数的检查 -->
        <module name="LocalVariableName" />
        <!-- 包名的检查（只允许小写字母），默认^[a-z]+(\.[a-zA-Z_][a-zA-Z_0-9_]*)*$ -->
        <!-- <module name="PackageName">
            <property name="format" value="^[a-z]+(\.[a-z][a-z0-9]*)*$" />
            <message key="name.invalidPattern" value="包名 ''{0}'' 要符合 ''{1}''格式."/>
        </module> -->
        <!-- 仅仅是static型的变量（不包括static final型）的检查 -->
        <module name="StaticVariableName" />
        <!-- Class或Interface名检查，默认^[A-Z][a-zA-Z0-9]*$-->
        <module name="TypeName">
             <property name="severity" value="warning"/>
             <message key="name.invalidPattern" value="名称 ''{0}'' 要符合 ''{1}''格式."/>
        </module>
        <!-- 非static型变量的检查 -->
        <module name="MemberName" />
        <!-- 方法名的检查 -->
        <module name="MethodName" />
        <!-- 方法的参数名 -->
        <module name="ParameterName " />
        <!-- 常量名的检查（只允许大写），默认^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$ -->
        <module name="ConstantName" />

        <!-- 定义检查 -->
        <!-- 检查数组类型定义的样式 -->
        <module name="ArrayTypeStyle"/>
        <!-- 检查long型定义是否有大写的“L” -->
        <module name="UpperEll"/>

        <!-- 长度检查 -->
        <!-- 每行不超过140个字符 -->
        <module name="LineLength">
            <property name="max" value="140" />
        </module>
        <!-- 方法不超过50行 -->
        <module name="MethodLength">
            <property name="tokens" value="METHOD_DEF" />
            <property name="max" value="60" />
        </module>
        <!-- 方法的参数个数不超过5个。 并且不对构造方法进行检查-->
        <module name="ParameterNumber">
            <property name="max" value="5" />
            <property name="ignoreOverriddenMethods" value="true"/>
            <property name="tokens" value="METHOD_DEF" />
        </module>

        <!-- 空格检查-->
        <!-- 方法名后跟左圆括号"(" -->
        <module name="MethodParamPad" />
        <!-- 在类型转换时，不允许左圆括号右边有空格，也不允许与右圆括号左边有空格 -->
        <module name="TypecastParenPad" />
        <!-- 检查在某个特定关键字之后应保留空格 -->
        <module name="NoWhitespaceAfter"/>
        <!-- 检查在某个特定关键字之前应保留空格 -->
        <module name="NoWhitespaceBefore"/>
        <!-- 操作符换行策略检查 -->
        <module name="OperatorWrap"/>
        <!-- 圆括号空白 -->
        <module name="ParenPad"/>
        <!-- 检查分隔符是否在空白之后 -->
        <module name="WhitespaceAfter"/>
        <!-- 检查分隔符周围是否有空白 -->
        <module name="WhitespaceAround"/>

        <!-- 修饰符检查 -->
        <!-- 检查修饰符的顺序是否遵照java语言规范，默认public、protected、private、abstract、static、final、transient、volatile、synchronized、native、strictfp -->
        <module name="ModifierOrder"/>
        <!-- 检查接口和annotation中是否有多余修饰符，如接口方法不必使用public -->
        <module name="RedundantModifier"/>

        <!-- 代码块检查 -->
        <!-- 检查是否有嵌套代码块 -->
        <module name="AvoidNestedBlocks"/>
        <!-- 检查是否有空代码块 -->
        <module name="EmptyBlock"/>
        <!-- 检查左大括号位置 -->
        <module name="LeftCurly"/>
        <!-- 检查代码块是否缺失{} -->
        <module name="NeedBraces"/>
        <!-- 检查右大括号位置 -->
        <module name="RightCurly"/>

        <!-- 代码检查 -->
        <!-- 检查空的代码段 -->
        <module name="EmptyStatement"/>
        <!-- 检查在重写了equals方法后是否重写了hashCode方法 -->
        <module name="EqualsHashCode"/>
        <!-- 检查局部变量或参数是否隐藏了类中的变量 -->
        <module name="HiddenField">
            <property name="tokens" value="VARIABLE_DEF"/>
        </module>
        <!-- 检查子表达式中是否有赋值操作 -->
        <module name="InnerAssignment"/>
        <!-- 检查是否有"魔术"数字 -->
        <module name="MagicNumber">
           <property name="ignoreNumbers" value="0, 1"/>
           <property name="ignoreAnnotation" value="true"/>
       </module>
        <!-- 检查switch语句是否有default -->
        <module name="MissingSwitchDefault"/>
        <!-- 检查是否有过度复杂的布尔表达式 -->
        <module name="SimplifyBooleanExpression"/>
        <!-- 检查是否有过于复杂的布尔返回代码段 -->
        <module name="SimplifyBooleanReturn"/>

        <!-- 类设计检查 -->
        <!-- 检查类是否为扩展设计l -->
        <!-- 检查只有private构造函数的类是否声明为final -->
        <module name="FinalClass"/>
        <!-- 检查接口是否仅定义类型 -->
        <module name="InterfaceIsType"/>
        <!-- 检查类成员的可见度 检查类成员的可见性。只有static final 成员是public的
        除非在本检查的protectedAllowed和packagedAllowed属性中进行了设置-->
        <module name="VisibilityModifier">
            <property name="packageAllowed" value="true"/>
            <property name="protectedAllowed" value="true"/>
        </module>

        <!-- 语法 -->
        <!-- String的比较不能用!= 和 == -->
        <module name="StringLiteralEquality"/>
        <!-- 限制for循环最多嵌套2层 -->
        <module name="NestedForDepth">
            <property name="max" value="2"/>
        </module>
        <!-- if最多嵌套3层 -->
        <module name="NestedIfDepth">
            <property name="max" value="3"/>
        </module>
        <!-- 检查未被注释的main方法,排除以Appllication结尾命名的类 -->
        <module name="UncommentedMain">
            <property name="excludedClasses" value=".*[Application,Test]$"/>
        </module>
        <!-- 禁止使用System.out.println -->
        <module name="Regexp">
            <property name="format" value="System\.out\.println"/>
            <property name="illegalPattern" value="true"/>
        </module>
        <!-- return个数 3个-->
        <!-- <module name="ReturnCount">
            <property name="max" value="3"/>
        </module> -->
        <!--try catch 异常处理数量 3-->
        <module name="NestedTryDepth ">
            <property name="max" value="3"/>
        </module>
        <!-- clone方法必须调用了super.clone() -->
        <module name="SuperClone" />
        <!-- finalize 必须调用了super.finalize() -->
        <module name="SuperFinalize" />


    </module>
</module>

```
新建一个文件，命名可以任意，然后在插件配置中，添加该模板文件即可：
![](/images/checkstyle3.jpeg)

### 2. 常见的代码规范

#### 2.1 命名风格

1. 代码中的命名均不能以下划线或美元符号开始，也不能以下划线或美元符号结束。
2. 类名使用`UpperCamelCase`风格。
3. 方法名、参数名、成员变量、局部变量都统一使用`lowerCamelCase`风格，必须遵从 驼峰形式。
4. 常量命名全部大写，单词间用下划线隔开，力求语义表达完整清楚，不要嫌名字长。
5. POJO 类中布尔类型的变量，都不要加`is`前缀，否则部分框架解析会引起序列化错误。
反例:定义为基本数据类型Boolean isDeleted的属性，它的方法也是isDeleted()，RPC框架在反向解析的时候，“误以为”对应的属性名称是 deleted，导致属性获取不到，进而抛出异常。
6. 包名尽量使用小写，但有时会出现多个单词组合的情况，那么也使用`lowerCamelCase`风格即可。包名统一使用单数形式，但是类名如果有复数含义，类名可以使用复数形式。
7. 接口类中的方法和属性不要加任何修饰符号(public也不要加)，保持代码的简洁性，并加上有效的`Javadoc`注释。尽量不要在接口里定义变量，如果一定要定义变量，肯定是与接口方法相关，并且是整个应用的基础常量。
8. 【参考】Service/DAO层方法命名规约:
> * 获取单个对象的方法用get做前缀。
> * 获取多个对象的方法用list做前缀，复数形式结尾如:listObjects。
> * 获取统计值的方法用count做前缀。
> * 插入的方法用save/insert做前缀。
> * 删除的方法用remove/delete做前缀。
> * 修改的方法用update做前缀。

#### 2.2 常量定义

1. 不允许任何魔法值(即未经预先定义的常量)直接出现在代码中。
2. 在 long 或者 Long 赋值时，数值后使用大写的 L，不能是小写的 l，小写容易跟数字 1 混淆，造成误解。
3. 不要使用一个常量类维护所有常量，要按常量功能进行归类，分开维护。
4. 如果变量值仅在一个固定范围内变化用 enum 类型来定义。

#### 2.3 代码格式

1. 基本的格式示例
```java
public static void main(String[] args) {
    // 缩进 4 个空格. 注释的双斜线与注释内容之间有且仅有一个空格。
    String say = "hello";
    // 运算符的左右必须有一个空格
    int flag = 0;
    // 关键词 if 与括号之间必须有一个空格，括号内的 f 与左括号，0 与右括号不需要空格
    if (flag == 0) {
        System.out.println(say);
    }
    // 左大括号前加空格且不换行;左大括号后换行
    if (flag == 1) {
        System.out.println("world");
    // 右大括号前换行，右大括号后有 else，不用换行
    } else {
        System.out.println("ok");
    // 在右大括号后直接结束，则必须换行
    }
}
```
2. 单行字符数限制不超过 120 个，超出需要换行
```java
StringBuffer sb = new StringBuffer();
// 超过 120 个字符的情况下，换行缩进 4 个空格，点号和方法名称一起换行
sb.append("zi").append("xin")...
    .append("huang")...
    .append("huang")...
    .append("huang");
```
3. 方法参数在定义和传入时，多个参数逗号后边必须加空格。
4. 【推荐】单个方法的总行数不超过 80 行。
5. 【推荐】不同逻辑、不同语义、不同业务的代码之间插入一个空行分隔开来以提升可读性。

#### 2.4 OOP 规约
1. 所有的覆写方法，必须加@Override 注解。
2. Object 的 equals 方法容易抛空指针异常，应使用常量或确定有值的对象来调用 equals。如`"test".equals(object);`
3. 所有的相同类型的包装类对象之间值的比较，全部使用`equals`方法比较。对于 `Integer var = ?` 在-128 至 127 范围内的赋值，Integer 对象是在 IntegerCache.cache 产生，会复用已有对象，这个区间内的 Integer 值可以直接使用==进行 判断，但是这个区间之外的所有数据，都会在堆上产生，并不会复用已有对象，这是一个大坑， 推荐使用 equals 方法进行判断。
4. 关于基本数据类型与包装数据类型的使用标准如下:
> * 【强制】所有的POJO类属性必须使用包装数据类型。POJO 类属性没有初值是提醒使用者在需要使用时，必须自己显式地进行赋值，任何NPE 问题，或者入库检查，都由使用者来保证。数据库的查询结果可能是 null，因为自动拆箱，用基本数据类型接收有 NPE 风险。
> * 【强制】RPC方法的返回值和参数必须使用包装数据类型。
> * 【推荐】所有的局部变量使用基本数据类型。
5. 定义 DO/DTO/VO 等 POJO 类时，不要设定任何属性默认值。POJO 类的 createTime 默认值为 new Date()，但是这个属性在数据提取时并没有置入具体值，在更新其它字段时又附带更新了此字段，导致创建时间被修改成当前时间。
6. POJO 类必须写 toString 方法。使用 IDE 中的工具:source> generate toString 时，如果继承了另一个 POJO 类，注意在前面加一下 super.toString。不要使用JSON工具类实现，因为json序列化有性能消耗。

#### 2.5 日志
调试时使用的日志，上线前必须删除。
关键逻辑必须记录日志，并包含相关参数信息。写下这条日志时，要清楚其作用，怎么定位，能用来挽救谁的性命。
日志需要控制级别，大量地输出无效日志，不利于系统性能提升，也不利于快速定位错误点。
统一使用slf4j日志框架，不要使用加号拼接参数，而是使用占位符。
```Java
logger.debug("Processing trade with id: {} and symbol : {} ", id, symbol);
```
