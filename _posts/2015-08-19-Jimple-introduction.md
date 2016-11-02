---
published: true
layout: post
title: Jimple--Java程序分析基础
category: program
tags: 
  - java
time: 2015.08.19 14:22:00
excerpt: 用户的Java代码被编译为class文件后才能在JVM中执行，Jimple是Java到class之间的一个代码形式，Soot提供了对Jimple代码的操作，可以很方便的修改class字节码。

---

## 1. Jimple是什么

Jimple是由soot生成的，Java Source文件到Java Class文件的中间文件。Jimple文件相比Class文件更加容易阅读，是soot进行Java字节码操作的对象。
要对Jimple操作，首先需要实例化一个Jimple的环境对象scene :

```scala
val scence = Scene.v() 
```
    
然后，再对scene进行分析和操作。

---

## 2. Jimple的组成

Jimple，或者说scene中，有与Java相对应的类Class，字段Field，方法Method等基本概念；还有Jimple自定义的Unit，Box，Value，CallGraph等概念。

### Scene

scene中可以直接获取Java Source中的所有Class，每一个Class都会生成一个Jimple文件与之对应。如果是内部类，名称前加其所在类的名称并用$符号连接。

```java
org.apache.spark.flint.demo.DenseVectorJavaLR
org.apache.spark.flint.demo.DenseVectorJavaLR\$DataPoint
```

### Class

class代表一个Java Class类，相应的通过Jimple可以获取class中包含的字段Field，方法Method，甚至是class的父类方法superClass，继承的接口Interface等。
根据Java的定义，每一个类会包含有构造器，因此class的methods中一定会有一个构造方法，方法名为“&lt;init>”，参数与构造方法的参数相同。同时，在主类mainClass，即包含有main入口方法的类中，如果定义有全局变量，虽然这也属于Field，但是这些变量的初始化操作在一个新的方法中执行，方法名为“&lt;clinit>”。

```java
class generated.org.spark.flint.demo.DenseVectorJavaLR extends java.lang.Object
```

### Field

Class中的字段即Field，字段被引用或者赋值时，是通过FieldRef实现的。FieldRef中包含了declaringClass，name等基本信息。在class中通过`getFieldByName(String name)`获取字段时，如果没有该名字的字段，Jimple会自动生成一个。

```java
double x;
double[] y;
```

### Method

方法Method的组成有参数ParameterList，返回类型ReturnType，方法体Body。Method的Body是分析和修改Method的基础，包含了Method中所有的语句单元Unit。Method也可以被调用，调用的方法为MethodRef。同样，在通过`getMethodByName(String name)`时，如果名字、参数、返回类型有一个不同，Jimple会自动生成一个，并且该方法体中有Error。

```java
void <init>(double[],double)
void dot(DenseVector)
```

### Body & Unit

Body代表了一个Method的方法体，Body中定义有method的本地变量Local和执行语句Unit。执行语句Unit的类型Stmt常见的有（详细分析见Box）:

+ 定义语句 JIdentityStmt

```java
r1 := @parameter0: double[];
```

+ 赋值语句 JAssignmentStmt

```java
r0.<org.apache.spark.flint.demo.DenseVectorJavaLR$DataPoint: double[] x> = r1;
```

+ 引用语句 JInvokeStmt

```java
specialinvoke r0.<java.lang.Object: void <init>()>();
```

+ 返回语句 JReturnStmt

```java
return $r3;
```

### Box & Value

Box是Unit的组成元素，每一个元素都需要放在Box中，例如

+ 定义语句JIdentityStmt包含有两个Box，左侧的一般为JimpleLocal类型，表示一个本地变量，右侧为Ref引用，有两种ThisRef和ParameterRef，前者指向当前类，后者指向method的参数，左右两侧通过“:=” 符号连接。

```java
r1 := @parameter0: double[];
```

+ 赋值语句JAssignmentStmt同样是左右结构，通过“=”符号连接。但是，leftOpBox或者rightOpBox可能还嵌套有多个Box，因为FieldRef和MethodRef提供指向的对象时还会提供参数值，参数值也放置在Box中。除了下面例子中的virtualinvoke外，还有JNewExpr、JCastExpr等常见例子。

```java
$z0 = virtualInvoke $r4.<java.lang.String boolean equals(java.lang.Object)>("Double");
```

+ 引用语句JInvokeStmt引用一个方法，由MethodRef所在的invokeBox和提供参数的Box组成。引用语句还有specialinvoke、interfaceinvoke和virtualinvoke之分。可以根据需要进行转换。

```java
specialinvoke r0.<java.lang.Object: void <init>()>();
```

---

## 3. Jimple可以做什么

Jimple可以让我们方便的分析Java字节码，这在编译优化中是非常有用的。静态分析利用上面的Jimple部件就可以分析、修改；动态分析则需要利用Jimple的CallGraph，后续再继续学习利用Jimple做动态分析。利用Jimple做静态分析，可以：

- 裁减一个复杂的类，将不需要的字段、方法、方法中的变量删除

- 组装方法

- 泛型特化

- 对象特性分析

- 修改代码流程（这比源代码层面复杂很多，但是却是经常会遇到的）

 
