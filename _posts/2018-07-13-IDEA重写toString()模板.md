---
title: IDEA重写toString()模板
date: 2018-07-13 17:41:50
tags: idea
categories: idea
---

IDEA中默认生成的toString()方法使用加号拼接，并且加号在每一行的末尾，导致checkstyle检查不通过，并且，如果频繁打印日志，也会产生大量字符串常量。
下面通过重写toString()方法，将其改为StringBuilder拼接Json。

<!-- more -->

首先在某个类中输入快捷键`Ctrl+Enter`，弹出的Generate对话框中选择toString()，然后选择Settings：
![](/images/idea1.jpeg)
在模板中点击新增：
![](/images/idea2.jpeg)
名字随便起，内容填写：
```Java
public java.lang.String toString() {
final java.lang.StringBuilder sb = new java.lang.StringBuilder("$classname{");
#set ($i = 0)
#foreach ($member in $members)#if ($i == 0)
sb.append("#####
#else
sb.append(",####
#end#if ($member.string || $member.date)
\"$member.name\":\"")
#else
\"$member.name\":")
#end#if ($member.primitiveArray || $member.objectArray)
.append(java.util.Arrays.toString($member.name));
#elseif ($member.string || $member.date)
.append($member.accessor).append('\"');
#else
.append($member.accessor);
#end#set ($i = $i + 1)
#end
sb.append('}');
return sb.toString();
}
```
