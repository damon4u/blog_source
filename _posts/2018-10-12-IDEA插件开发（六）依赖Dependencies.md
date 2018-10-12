---
title: IDEA插件开发（六）依赖Dependencies
date: 2018-10-12 17:41:50
tags: idea
categories: [idea,java]
---

你的插件可能会依赖其他插件，内置的、第三方的或者你自己的。为了引入这些依赖，需要执行如下步骤：
* 如果插件时内置的，那么启动沙箱环境，在里面安装插件。
* 在 __IntelliJ Platform SDK__ 中添加插件的jar包。打开 __Project Structure__ 对话框，选择使用的SDK，点击加号选择要添加的插件jar包。如果是内置插件，那么插件jar包在主安装目录下的`plugins/<pluginname>`或者`plugins/<pluginname>/lib`目录下。如果是非内置插件，那么插件jar包在`Sandbox Home`指定的目录下的`config/plugins/<pluginname>`或者`config/plugins/<pluginname>/lib`内。
* 添加`<depends>`标签到`plugin.xml`中，将插件的ID作为标签值。如果不知道插件的ID，可以到指定插件的`plugin.xml`中查看。

<!-- more -->

例如，我们的插件依赖`com.intellij.database`插件，那么我们需要将jar引入：
![](/images/idea-plugin16.png)

然后在`plugin.xml`中声明：
```xml
<depends optional="true">com.intellij.database</depends>
```
`optional`代表如果此插件没有安装，那么可以用，不过可能影响部分功能。

参考资料：
[Plugin Services](https://www.jetbrains.org/intellij/sdk/docs/basics/plugin_structure/plugin_services.html)
