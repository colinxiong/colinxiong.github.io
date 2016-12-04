---
published: true
layout: post
title: Zookeeper详解（一）：Client端
category: distributed system
tags: 
  - 分布式系统
time: 2016.12.04 14:23:00
excerpt: Zookeeper是Apache中维护分布式系统一致性的框架，Client端由用户建立，发送请求到Server端。

---

## 1. Client端

### 1.1 创建Client

Client 端实例化一个Zookeeper对象作为起点，实际的构造器如下：

```java
public Zookeeper(String connectString, 
      int sessionTimeout, 
      Watcher watcher, 
      boolean canBeReadOnly, 
      HostProvider aHostProvider, 
      ZKClientConfig clientConfig){
    //...
}
```
当然，Zookeeper提供了多个构造器，但是用户至少需要指定三个对象：connectString，用户需要连接的Server的地址；sessionTimeout，用户指定的连接超时的时间；watcher，用户提交请求后对可能出现的事件的处理方法。

Watcher究竟是什么？从源码上看，Watcher是一个接口，用户需要自定义一个类实现Watcher，具体是实现其中的process方法。process方法的参数是一个WatchedEvent，包含了发生的事件类型、Zookeeper当前的状态和事件发生的路径。因此，Watcher实际上是一个用来处理Zookeeper事件的观察者，由用户决定发生不同事件时该如何处理。事件类型包括：新建Node，删除Node等。

### 1.2 启动

Zookeeper构造器内部实际上初始化的对象包括：

- WatchManager。管理用户指定的Watcher。
- HostProvider。根据connectString中的server地址而创建的一个管理连接的server的对象，默认实例化为StaticHostProvider，会对初始的server随机洗牌。
- ClientCnxn。真正重要的对象！

ClientCnxn的构造器如下：

```java
public ClientCnxn(String chrootPath,
      HostProvider hostProvider,
      int sessionTimeout,
      Zookeeper zooKeeper,
      ClientWatchManager watcher,
      ClientCnxnSocket clientCnxnSocket,
      long sessionId,
      byte[] sessionPasswd,
      boolean canBeReadOnly){
    // ...
}
```

其中包含了Zookeeper对象及它所创建的一些对象。而其中最为核心的就是启动了两个线程：SendThread和EventThread。

#### 1.2.1 SendThread

SendThread需要传入一个ClientCnxnSocket对象，而ClientCnxnSocket对象是创建与server的socket连接、发送请求数据、接收数据的关键网络接口。实例化时可以有NIO和Netty两种方式（后续深入学习下NIO和Netty之间的区别）。

SendThread线程内部是一个死循环，当client是alive的话循环执行操作，这些操作包括：

- 检查连接的状态
- 检查Client的状态
- 检查Zookeeper的状态
- 关键是执行ClientCnxnSocket的doTransprot方法。

ClientCnxnSocket的doTransprot方法如下：

```java
// NIO
void doTransport(int waitTimeOut,
      List<Packet> pendingQueue,
      ClientCnxn cnxn){
    // update channel and connection
    // ...
    doIO(pendingQueue, cnxn);
}

void doIO(List<Packet> pendingQueue, ClientCnxn cnxn){
    // if readable
    // read data
    // ...
    // if writeable
    // write pending packet
    // ...
}

// Netty
void doTransprot(int waitTimeOut,
      List<Packet> pendingQueue,
      ClientCnxn cnxn){
      // wake up connection
      // ...
      doWrite(pendingQueue, head, cnxn);
}

private void doWrite(List<Packet> pendingQueue, Packet p, ClientCnxn xncn){
      // send packet
      // ...
}
```

所以，SendThread实际上就是循环读取Socket连接中的数据，和循环发送等待队列中的Packet。内部实现完全是Java的Socket编程。

那么，问题在于发送的packet是什么？什么时候加入等待队列？接收的数据是什么？如何处理接收的数据？

#### 1.2.2 EventThread

EventThread是一个守护进程，内部的死循环从Event队列中取出Event，根据类型进行相应的处理，最终会传递给用户定义的Watcher中的process方法。

同样问题在于Event何时添加进入队列？由于EventThread比较简单一些，所以这里这里直接得出以下添加Event到队列的时间点：

- 启动ClientCnxnSocket连接时。
- 获取Server返回的验证信息时。
- 获取Server返回的通知消息时。
- 获取Server返回的断开连接消息时。

### 1.3 发送请求

在用户定义了Zookeeper对象后，内部就已经启动完Client了，重点是SendThread。现在我们需要关注它其中的队列是和何时被填充的，何时被取出发送到Server端的，何时从Server收到回复，收到的回复如何处理。

用户建立请求的方法是Zookeeper提供的，也就是说用户只需要调用Zookeeper的方法接口就可以了。具体包括：

```java
public String create(...);
public void delete(...);
public Stat exists(...);
public byte[] getData(...);
public Stat setData(...);
public Stat setACL(...);
public List<ACL> getACL(...);
public void sync(...);
public void multi(...);
```

这些方法内部就会生成一个请求，封装到Packet中，添加到SentThread的队列中。这就是SentThread中Packet队列的来源，由用户在代码中定义。

具体看Packet是如何包装各种不同的请求的！Packet的数据组成如下：

```java
RequestHeader requestHeader;
ReplyHeader replyHeader;
Record request;
Record response;
//...
```

这里面有一个基本的接口Record，大部分相关类都实现了该接口，实现内部的serialize和deserialize方法得到可以用于网络传输的Bytes序列。

- RequestHeader内部有两个参数：xid，type。type由调用的方法决定。xid用于按顺序的编号。
- ReplyHeader内部有三个参数：xid，zxid，err。xid用于匹配顺序编号，保证Request操作的顺序性。zxid是Zookeeper分配的操作号。当处理错误时err会被定义。
- request则是调用的方法根据请求类型建立的不同类，但是这些类都实现了Record接口。比如CreateRequest，DeleteRequest。Request内部携带了不同的请求所需要的数据。
- response和request对应，什么请求就返回什么回复。比如CreateResponse，DeleteResponse。

因此，所有的用户请求都以Packet的形式提交到SendThread的队列。那么，Packet序列化为网络的bytes序列时组成是什么？答案如下，只有RequestHeader和request被序列化到了bytes序列中。

RequestHeader | request | 
:---:|:---:
int xid, int type | according to the request type

### 1.4 接收回复

SendThread在发送请求的同时还会接收Session中Server端传输过来的Response，加入pendingQueue。然后调用readResponse处理。

首先，反序列化出ReplyHeader，根据ReplyHeader中的xid判断：

- -1：Server发送的Event需要处理。
- -2：ping操作。
- -4：验证操作。
- 其他：处理正常请求的回复。

其次，需要保证requestHeader的xid与replyHeader的xid一致。

最后，根据replyHeader的zxid更新最后处理的zxid。

至此，Client端的整个流程以及流程中重要的数据类型的格式都已经很详细了。更加需要熟悉的是Client端实际传输给Server的数据格式。
