---
published: true
layout: post
title: Java JNI 使用与要点
category: program
tags: 
  - cpp
  - java
time: 2018.04.24 21:03:00
excerpt: Java支持使用JNI加载cpp编写的so文件，使用关键字native注释加载的方法。在使用时不仅要注意运行JVM的机器的环境，还需要注意多线程调用可能出现的问题。

---


当我们需要在Java程序中调用某个C++函数时，就需要通过JNI调用编译好的so文件，并在java代码中加载和使用对应so文件中的函数。由于jvm支持不同环境，而cpp与编译内核有关，所以还需要考虑环境因素。在多线程中调用so文件还需要考虑到线程安全。

## JNI使用方法

### Java中注册需要使用的方法

主要是将需要使用JNI load的方法注册为native。

```
package com.test.jni

public class JNITest {
    long myId = 0;
    public native long getId(long yourId);
} 
```

### cpp中编写和编译相应的方法

```
JNIEXPORT long JNICALL Java_com_test_jni_JNITest_getId(JNIEnv *env, jclass j_class, jlong yourId) {  
    env->GetJavaVM(&gs_jvm);    
    long process_id = 0;
    // process function to processId
    // ...
    return process_id;  
} 
```

编译成libjnitest.so文件后放置在项目的resources文件夹下可以通过直接路径/加载。

### 在Java的class中加载so文件

```
InputStream in = JNITest.class.getResourceAsStream("/libjnitest.so");
File soFile = File.createTempFile("libjnitest", ".so");
Files.copy(in, soFile.toPath(), StandardCopyOption.REPLACE_EXISTING);
in.close();
System.load(soFile.toString);
```
以上代码段从so文件中加载native方法，每个JVM只会load一次，因此建议放在static代码块中执行。

## 使用注意事项

## 执行环境

Java程序的执行环境只与JVM有关，但是so文件的执行则与机器有关。需要执行Java程序的机器与编译so文件的机器的c++编译版本等相同。

## 多线程

如果Java代码中有多线程调用同一个对象的native方法（尤其是在分布式计算框架，如Spark，中调用JNI的方法），会报错：

```
*** Error in 'java': double free or corruption (out): 0x00007f665c04a830 ***
```

出现这种问题的原因就是多线程同时访问native方法造成的。解决方法有两种：

 > 1. native方法用synchronized关键字修饰。
 > 2. 在cpp中实现线程安全的访问，可以参考后续的参考文献解决。

---

*本文参考：<https://blog.csdn.net/lovingprince/article/details/2793504>*
*本文参考：<https://stackoverflow.com/questions/22491797/java-double-free-or-corruption>*
