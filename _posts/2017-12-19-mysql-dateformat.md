---
title: date_format函数
date: 2017-12-19 17:41:50
tags: mysql
categories: 数据库
---

### 1.定义和用法

`DATE_FORMAT()`函数用于以不同的格式显示日期/时间数据。

<!--  more -->

### 2.语法

```
DATE_FORMAT(date,format)
```
`date`参数是合法的日期。`format`规定日期/时间的输出格式。

|  格式  |  描述 |
| -----  | -----  |
|%Y|	年，4 位|
|%y|	年，2 位|
|%m|	月，数值(00-12)|
|%b|	缩写月名(Dec)|
|%c|	月，数值|
|%D|	带有英文前缀的月中的天|
|%d|	月的天，数值(00-31)|
|%e|	月的天，数值(0-31)|
|%j|	年的天 (001-366)|
|%H|	小时 (00-23)|
|%h|	小时 (01-12)|
|%I|	小时 (01-12)|
|%k|	小时 (0-23)|
|%l|	小时 (1-12)|
|%i|	分钟，数值(00-59)|
|%S|	秒(00-59)|
|%s|	秒(00-59)|
|%f|	微秒|
|%U|	周 (00-53) 星期日是一周的第一天|
|%u|	周 (00-53) 星期一是一周的第一天|
|%p|	AM 或 PM|
|%a|	缩写星期名|

### 3.使用实例

统计每天0点到1点的数量
```
select date(create_time),count(1) from live where date_format(create_time, '%H:%i:%S')>'00:00:00' and date_format(create_time, '%H:%i:%S')<'01:00:00' group by date(create_time) order by create_time desc;
```
