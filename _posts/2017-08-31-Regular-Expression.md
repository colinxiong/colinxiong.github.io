---
published: true
layout: post
title: 正则表达式的使用
category: 编程基础
tags: 
  - scala
  - java
time: 2017.08.31 23:37:00
excerpt: 在生产应用中，需要处理很多结构化的数据，当读入到系统中做处理时，一条记录就是一个不同字段拼接而成的字符串，这时如果单纯使用字符串的split方法和分隔符来处理数据，不容易让人联想到这些数据在表中的格式组成。正则匹配可以让人非常直观的获取数据格式，对于代码审查来说也非常容易。

---

## 1. 基础正则表达式

### 1.1 特殊字符

- `^`和`$`: 表示从字符串头/尾开始匹配，做全字符串匹配时一般不会用，但是做替换匹配时加上，则非常有用。

```scala
val pattern = Pattern.compile("^work") //其实就是全匹配
val input1 = "work" // true
val input2 = "wor" // false
val input3 = "working" // false

val replace = "112345"
val result1 = replace.replaceAll("[1-9]{2}", "Z") // "ZZZ"，正则项表示两个连续非零数字
val result2 = replace.replaceAll("^[1-9]{2}", "Z") // "Z2345"
val result3 = replace.replaceAll("[1-9]{2}$", "Z") // "1123Z"
```

- `*`，`+`和`?`：表示匹配表达式的次数，分别为匹配0-n次，匹配1-n次和匹配0-1次。

```scala
val pattern1 = Pattern.compile("work[1-9]*") // true，有0-n个1-9的数字
val pattern2 = Pattern.compile("work[1-9]?") // false，有0-1个1-9的数字
val pattern3 = Pattern.compile("work[1-9]+") // true，有1-n个1-9的数字
val input1 = "work12345"

```

- `{n}`，`{n,}`和`{n,m}`：表示匹配n次，至少匹配n次，至少匹配n次但至多匹配m次。

```scala
val pattern1 = Pattern.compile("work[1-9]{4}}") // true，有4个1-9的数字
val pattern2 = Pattern.compile("work[1-9]{5}") // false，有5个1-9的数字
val pattern3 = Pattern.compile("work[1-9]{2,}") // true，至少有2个1-9的数字
val pattern4 = Pattern.compile("work[1-9]{2,3}") // false，有2-3个1-9的数字
val pattern5 = Pattern.compile("work[1-9]{2,6}") // true，有2-6个1-9的数字
val input1 = "work12345"
```

### 1.2 功能字符

- `x|y`，`[xy]`和`[^xy]`：表示匹配x或y，匹配x和y，匹配不属于x和y。

- `\b`，`\B`，`\d`和`\D`（均需要转义）：表示匹配单词边界，匹配非单词边界，匹配一个数字，匹配一个非数字

```scala
val input = "HardWork eWork1234 eWorkHard"
println(input.replaceAll("rk\\b", "Z")) // HardWoZ eWork1234 eWorkHard
println(input.replaceAll("rk\\B", "Z")) // HardWork eWoZ1234 eWoZHard
println(input.replaceAll("rk\\d", "Z")) // HardWork eWoZ234 eWorkHard
println(input.replaceAll("rk\\D", "Z")) // HardWoZeWork1234 eWoZard，空格也属于非数字
println(input.replaceAll("rk[^\\d|\\s]", "Z")) // HardWork eWork1234 eWoZard，替换跟着的不是数字且不是空格的rk为Z
```

## 2. 应用

### 2.1 特征

特征是机器学习里非常常用的一个概念，一个特征往往有主键和副键之分，表示一个特征时用`{主键pk.副键sk}`的样子来表示，正则表达式如下：

```java
\\{.+\\..+\\}
```

### 2.2 交叉特征

特征之间做交叉是为了体现特征之间的关系而产生的新特征，例如点积交叉、乘积交叉等。表示为`{特征1}{交叉符号}{特征2}`，正则表达式如下：

```java
\\{.+\\..+\\}\\*\\{.+\\..+\\}
```