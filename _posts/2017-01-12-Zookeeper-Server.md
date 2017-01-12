---
published: true
layout: post
title: Zookeeper详解（二）：Server端
category: distributed system
tags: 
  - 分布式系统
time: 2017.01.12 11:15:00
excerpt: Zookeeper是Apache中维护分布式系统一致性的框架，多个Server端接收Client端请求，协同服务。

---

## 2. Server

Server端自然是Zookeeper的精髓所在，Server端主要有两个模块：Leader的选举，Packet的处理流程。当在每个节点启动Zookeeper的Server后，节点之间会进行选举，选出最终的LeaderZookeeperServer，其他的节点为FollowerZookeeperServer或者ObserverZookeeperServer。然后才能接受Client端发送过来的Packet请求。

### 2.1 启动

每一个节点上都可以通过ZookeeperServerMain启动当前节点上的ZookeeperServer进程，但这只是一个standalone的进程。当启动一组Sever用于提供服务时，这组Server被称为一个Quorum。QuorumServer是这一组Srever的父类，是ZookeeperServer的子类。这里不考虑standalone的启动方式，以Quorum启动为主。

Quorum中每个节点都需要通过QuorumPeerMain来启动，提供的参数是config的路径。启动时，先实例化QuorumPeerConfig来管理所有的配置，然后启动一个purge task来定期清除快照文件。其次，通过ServerCnxnFactory创建两个用于连接的对象ServerCnxnFactory。最后启动QuorumPeer。

QuorumPeer是一个线程对象，也是ZookeeperServer启动时最关键的对象，可以认为是一个Context。最终的构造器如下：

```java
public QuorumPeer(Map<Long, QuorumServer> quorumPeers,
        File dataDir,
        int electionType,
        long myid,
        int tickTime,
        int initLimit,
        int syncLimit,
        boolean quorumListenOnAllIPs,
        ServerCnxnFactory cnxnFactory,
        QuorumVerifier quorumConfig){
    // ...
}
```

QuorumPeer启动时，依次启动或者更新了三个重要的对象：

```java
public synchronized void start(){
    loadDataBase();
    startServerCnxnFactory();
    startLeaderElection();
    super.start();
    // ...
}
```

#### 2.1.1 加载ZKDatabase

ZKDatabase在QuorumPeer实例化时实例化，其构造器方法如下：

```java
public ZKDatabase(FileTxnSnapLog snapLog){
    dataTree = new DataTree();
    sessionsWithTimeouts = new ConcurrentHashMap<Long, Integer>();
    this.snapLog = snapLog;
}
```

ZKDatabase自己维护DataTree形成一个目录结构，用来存放元数据。创建时依次创建/，/zookeeper，/zookeeper/quota，/zookeeper/config目录。额外有一个ConcurrentHashMap<String,DataNode>用来快速检索树中的节点。

QuorumPeer调用ZKDatabase的loadBase方法，将数据从磁盘加载到内存中，以DataTree的形式进行管理。然后，添加一个认证的Proposal到自身的commitedLog中（后续会跟进该Proposal的作用）。

#### 2.1.2 启动ServerCnxnFactory

ServerCnxnFactory有两种实现方式，每种实现方式的启动如下：

- NIOServerCnxnFactory。启动一个acceptThread和一个expirerThread，以及一系列的selectorThread。
- NettyServerCnxnFactory。直接绑定端口即可。

ServerCnxnFactory的工作依赖ZookeeperServer，但是由于没有选举，这时ZookeeperServer还没有确定，等选举完成后才设置ZookeeperServer。ServerCnxnFactory的具体功能在接收请求时详解。

#### 2.1.3 开始Leader选举

开始选举时节点首先初始化自己的状态，启动UDP的Socket连接和一个ResponseThread，最后实例化选举算法，选举算法由QuorumPeer中的electionType决定，包括LeaderElection，AuthFastLeaderElection，FastLeaderElection等。本文以FastLeaderElection为对象分析。

目前为止的一系列动作都是在start()方法中启动的，启动完成后QuorumPeer开始执行run方法，执行选举操作。有一些启动的线程还没有详细分析，在后面的执行过程中再结合其作用分析。

### 2.2 选举

#### 2.2.1 状态

在QuorumPeer启动时，该节点的状态是初始状态LOOKING。随后通过选举可能会有如下的状态：

- LOOKING：正在选举中，未确定当前状态；
- LEADING：被选举为Leader，通知所有节点；
- FOLLOWING：被选举为Follower，回复Leader节点；
- OBSERVING：保持观察者状态，不参与选举。

#### 2.2.2 选票

选票Vote的组成如下：

```java
public class Vote{
    final private int version;
    final private long id;
    final private long zxid;
    final private long electionEpoch;
    final private long peerEpoch;
    final private ServerState state;
```

#### 2.2.3 选举的过程

QuorumPeer开始执行时，执行一个死循环，循环检测当前节点的状态State，根据State的内容做不同的操作：

- LOOKING：调用FastLeaderElection算法的lookForLeader方法。更新节点自己的ID，ZKDatabase中记录的最后Zxid，当前选举的代数Epoch。调用sendNotifications发送通知给所有其他的QuorumPeer，发送的消息包括建议的Leader（自己），建议的最后zxid，当前选举的代数等。然后处理接收到的其他QuorumPeer发送的通知：

  + nothing：超时后没有接受到消息继续发送通知
  + LOOKING：如果收到的通知满足以下三个条件之一：1）Epoch大于当前的Epoch；2）Epoch等于当前的Epoch但是Zxid大于当前的Zxid；3）当前的Epoch和Zxid都相同但是ServerID大于该节点的ServerID。更新建议的leader为通知中的leader。然后重新发送一个通知，通知中携带了自己将要投票的Leader（即建议的Leader）。由于每次都会记录自己处理的通知（这个通知实际上也是一个选票）都会记录下来，所以当记录中包含了所有的sever的选票后，停止选举。停止后，检查接收的消息队列中Leader有没有变化，只有没有变化时，才会更新自己的状态（如果proposalLeader等于自己，则自己为Leader，否则为Learner）。
  + FOLLOWING：不做任何事情
  + LEADING：确保所有的Server位于相同的Epoch。

- OBSERVING：与Leader通信同步，更新自己状态。
- FOLLOWING：与Leader通信同步，更新自己状态。
- LEADING：提供Leader服务，更新自己状态。

文字描述狠无力，随后补上图吧。


#### 2.2.4 选举过后

最终，每个Server都有自己归属的状态：Leading，Following，Observing。这些状态对应了自己在集群中的角色：Leader，Follower，Observer。下面看看它们各自如何从选举状态变为自己的角色：

- Leader：选举为Leader的Server，仅1个Leader。实例化自己节点上的leader变量：

```java
Leader leader = new Leader(this, 
    new LeaderZooKeeperServer(logFactory, this, this.zkDb));
```

Leader上需要：实例化一个LeaderZookeeperServer（设置ServerCnxnFactory的Server为该ZookeeperServer），加载ZKDatabase，启动LearnerCnxAcceptor接收Learner的消息，更新当前Epoche，与所有的Learner通信同步数据并检查心跳，没有个Learner都会对应一个LearnerHandler。最后，置当前QuorumPeer的Leader为空，因为这个Leader仅仅用于选举后初始化LeaderZookeeperServer。

- Follower：选举为Follower的Server。实例化自己节点上的follower变量：

```java
Follower follower = new Follower(this, 
    new FollowerZooKeeperServer(logFactory, this, this.zkDb));
```

Follower上需要：实例化一个FollowerZookeeperServer（继承自LearnerZookeeperServer，设置ServerCnxnFactory的Server为该ZookeeperServer），与Leader通信同时更新Epoch，同步数据。最后同样需要置当前QuorumPeer的Follower为空。

- Observer：与Follower相似，只是不需要更新选举的Epoch，因为它不参与选举；Observer也不参与写请求的投票（后文详细描述）。

### 2.3 Server提供服务

一个Quorum包含一个Leader，多个Follower或者Observer，它们一致对外提供服务，为了区分各个节点的功能，可以认为：在Quorum内部，有Leader，Learner（Follower和Observer）之分；在Quorum提供服务时，对Client来说，有LeaderZookeeperServer，LearnerZookeeperServer（FollowerZookeeperServer和ObserverZookeeperServer）之分。前者体现了内部节点一致性的根本，后者表示了对外提供服务的关键。

#### 2.3.1 接收请求

在Client端的请求数据封装为Packet提交到Server端，请求格式如下：

RequestHeader | request | 
:---:|:---:
int xid, int type | according to the request type

对于Server端来说，这就是接收到的请求的格式，负责接收请求的对象为ServerCnxnFactory（NIOServerCnxnFactory或者NettyServerCnxnFactory，这里以NIOServerCnxnFactory为例）和ServerCnxn。NIOServerCnxnFactory有AcceptThread和SelectorThread负责建立连接。AcceptThread通过select()方法循环检查每个连接的Session，通过doAccept()方法判断是否接收session连接。SelectorThread对接收的连接实例化NIOServerCnxn，然后监控session连接是否有读/写的数据，循环执行，通过handleIO()操作NIOServerCnxn处理缓冲区数据。

ZookeeperServer使用IOWorkRequest封装每个连接，将NIOServerCnxn封装到IOWorkRequest中，然后由WorkerService调度这些连接。调度到的WorkReuqest会启动一个新的线程来处理缓冲区数据。

调度到的WorkRequest被执行实际上是执行doWork()方法，也就是NIOServerCnxn的doIO()方法。doIO()方法包括了可读和可写两种状态的操作，接收请求涉及到可读的处理。

读取数据到incomingBuffer中，第一次连接时进行初始化操作readConnectRequest()；开始读取Request时，第一次读取的是4 letter word，也就是用来描述下一次要接受的数据的长度，随后要接收的才是Request的数据readRequest()，内部调用ZookeeperServer.processPacket()方法处理Request。

#### 2.3.2 预处理请求

接收的请求数据存放在incoming buffer中，数据格式已经介绍过了。所以处理的流程是：

- 反序列化出RequestHeader。Zookeeper中所有的Request和Packet都实现了Record接口，需要自己实现从Bytes序列中的序列化和反序列化方法。

```java
InputStream bais = new ByteBufferInputStream(incomingBuffer);
BinaryInputArchive bia = BinaryInputArchive.getArchive(bais);
RequestHeader h = new RequestHeader();
// 这种反序列化的方法还是比较简单的
h.deserialize(bia, "header");
```

- 根据RequestHeader中的type判断当前需要处理的请求的类型，决定如何处理。

请求的类型有很多种，主要分为两种：一种是由用户发起的请求，对ZKDatabase进行操作的请求；另一种是Server和Client端之间维护正常工作的请求。

文字描述这些请求（OpCode中描述了所有的请求类型）的处理方法如下：

- OpCode.auth：认证请求。不提交到执行流程中，认证完成后无论成功失败与否都直接回复Client。认证失败的同时还会关闭连接。
- Opcode.sasl：安全认证。是一种新的安全认证方式，也是直接回复给Client。
- 其他的请求做一个封装，封装为Request对象。这个Request对象是ZookeeperServer用来处理请求的实际对象，和Client发送过来的请求的数据格式是不同的，它的各个字段组成是：
  + cnxn：负责处理Request的当前Server上的ServerCnxn。
  + sessionId：Request数据来源的连接的ID。
  + cxid：client端发送该Request的序号ID，即传过来的RequestHeader的xid。
  + type：Request的类型，即传过来的RequestHeader中的type。
  + request：ByteBuffer，包含了用户提交的请求的数据，即incoming buffer中接收的Client传过来的request。
  + authInfo：认证消息，由Cnxn产生。
  + hdr：初始化可选，TxnHeader，消息头用于标识当前Request并记录时间
  + txn：初始化可选，Record，正式生成当前Request实际的操作
  + zxid：初始化可选但是后期必定更新，long，Request被处理的顺序序号ID

封装完成的Request被提交到正式的执行流程中，是Request被执行的具体流程。

#### 2.3.3 处理请求

当ZookeeperServer执行submitRequest(Request request)时，封装好的Request被提交到ZookeeperServer的处理流程中。而所有的ZookeeperServer都有一个firstProcessor，是submitRequest时第一个处理Request的Processor。所以首先可以联想到的是还会有很多个Processor，确实，有很多实现了RequestProcessor接口的Processor，而且整个执行流程就是由这些Processor组成的。有一些Processor是新的线程，通过生产者消费者模式从队列中取数据处理并传入下一个Processor；有些Processor就是一个普通的对象，处理完成后传入下一个Processor。所谓的传入下一个Processor就是调用Processor的processRequest()方法。

这时我们要分开考虑Leader和Follower，Observer上的Processor及流程的不同了，而且读请求、写请求、ping操作请求都是不同的。所以接下来通过新的一节2.4来描述。

#### 2.3.4 回复Client

Reqeust中有Cnxn对象时，表明属于当前的ServerCnxn接收的请求，因此也由它负责回复。回复的格式是RelyHeader + Record，Record是根据不同的请求实例化的不同回复对象。

### 2.4 Processor处理请求流程

读请求和写请求的处理流程不同，Leader和Follower协同工作，请求处理的流程也不相同。ZookeeperServer在确定自己角色后的方法中调用startup()启动zkServer，同时初始化Processor的初始化。

#### 2.4.1 Server上的Processor初始化

Leader上维护两个字段CommitProcessor和PrepRequestProcessor。建立的Processor处理链如下：

firstProcessor：LeaderRequestProcessor → PrepRequestProcessor(Thread) → ProposalRequestProcessor → CommitProcessor(Thread) → ToBeAppliedRequestProcessor → FinalRequestProcessor

而在ProposalRequestProcessor内部会维护一个SyncRequestProcessor，包含一条新的Processor链：SyncRequestProcessor(Thread) → AckRequestProcessor。这个Processor链并没有与Leader上的链连接起来。

Learner上会维护两个字段CommitProcessor和SyncRequestProcessor。Follower和Observer继承自Learner。

Follower上建立的Processor链如下：

firstProcessor：FollowerRequestProcessor(Thread) → CommitProcessor(Thread) → FinalRequestProcessor

同时Follower上也会有一个额外的Processor链：SyncRequestProcessor(Thread) → SendAckRequestProcessor

Observer上建立的Processor链如下：

firstProcessor：ObserverRequestProcessor(Thread) → CommitProcessor(Thread) → FinalRequestProcessor

Observer上默认开启了落盘机制，也就是会实例化一个SyncRequestProcessor，没有与上述Processor链连接。

#### 2.4.2 请求分类

所有提交到Server上的执行流程的请求有如下的分类（可以从PrepRequestProcessor中得出）：

读请求包括：

- OpCode.exists
- OpCode.getData
- OpCode.getACL
- OpCode.getChildren
- OpCode.getChildren2
- OpCode.ping
- OpCode.setWatches
- OpCode.removeWatches

写请求包括：

- OpCode.sync
- OpCode.createContainer, OpCode.create, OpCode.create2：创建新的数据记录
- OpCode.createTTL：
- OpCode.deleteContainer, OpCode.delete：
- OpCode.setData:
- OpCode.reconfig
- OpCode.setACL
- OpCode.check
- OpCode.multi
- OpCode.createSession, OpCode.closeSession

#### 2.4.3 Leader上的读请求处理

LeaderServer收到读请求后提交到firstProcessor，即LeaderRequestProcessor处理，各个Processor的处理如下：

- LeaderRequestProcessor：检查是否是本地的请求，是则创建一个虚拟的节点和会话连接，否则直接提交到下一个Processor。
- PrepRequestProcessor：添加请求到LinkedBlockingQueue中，线程执行一个死循环，从阻塞队列中取出队首的请求处理。检查会话连接，读请求的Hdr和Txn置为null。更新请求的zxid为当前Server的zxid。
- ProposalRequestProcessor：读请求直接发送到下一个processor处理。
- CommitProcessor：请求添加到阻塞队列queuedRequests中等待处理（其中的Request暂称为queuedRequest），线程执行一个死循环，等到queuedRequests中有Request时取出。读请求的话，检查request所属的session连接对应的pendingRequests（这是一个LinkedList类型）中是否有数据，如果有，将读请求添加到末尾，否则直接传递到下一个Processor。同时，顺序检查pendingRequests的Request，如果是读请求，依次传递到下一个Processor直到pendingRequest空或者第一个请求是写请求。请求的传递需要通过workpool调度，异步执行下保证每次只传递一个request到下一个request。
- ToBeAppliedRequestProcessor：读请求直接传递给下一个Processor。
- FinalRequestProcessor：在ZKDatabase中执行请求，根据Request中的Cnxn回复结果给Client。

#### 2.4.4 Follower上的读请求处理

FollowerServer收到读请求后提交到firstProcessor，即FollowerRequestProcessor处理，各个Processor的处理如下：

- FollowerRequestProcessor：检查是否是本地请求，是则创建一个虚拟的节点和会话连接。将request添加到LinkedBlockingQueue队列queuedRequest中。线程执行一个死循环，从queuedRequest中取出request，读请求直接传送到下一个processor。
- CommitProcessor：处理同Leader。
- FinalRequestProcessor：处理同Leader。

Observer上的读请求与Follower上的读请求处理流程相同。

可以看出，读请求在Follower和Leader上的步骤是基本类似的，FinalRequestProcessor之前的处理都没有很难的地方，统一在FinalRequestProcessor中执行ZKDatabase的读操作，然后由ServerCnxn回复Client。

#### 2.4.5 Leader上的写请求处理

写请求需要同步更新Leader和Learner上的ZkDatabase，所以会存在两者之间的交互。落盘到ZkDataBase的步骤在SyncRequestProcessor中执行。

Leader上接收到写请求，进入Processor处理链的流程如下：

- LeaderRequestProcessor：检查是否是本地的请求，是则创建一个虚拟的节点和会话连接，否则直接提交到下一个Processor。（与读请求相同）

- PrepRequestProcessor：添加请求到LinkedBlockingQueue中，线程执行一个死循环，从阻塞队列中取出队首的请求处理。检查会话连接，写请求需要初始化Hdr和Txn，Hdr对应TxnHeader，记录了request所属的会话连接、client的ID、Server上的zxid、当前时间、操作类型；Txn根据操作的类型实例化为不同的对象。随后，检查内存中的ZKDatabase是否包含该路径的数据节点，合法则获取父节点及当前节点，将父节点和当前节点分别生成的ChangeRecord添加到ZKDatabase的outstanding队列中。然后添加Request到下一个Processor。

- ProposalRequestProcessor：直接传递给下一个processor，写请求同时推送给所有的Learner节点，然后添加到另外一条Processor处理链的起始SyncRequestProcessor中。推送时，由Leader调用propose()方法，通过一个QuorumPacket对象（类型为Leader.PROPOSAL）发送。然后，生成一个Proposal对象，包含packet和request，添加到Leader上的outstandingProposals中。

  + SyncRequestProcessor：添加Request到阻塞队列queuedRequest中，线程执行一个死循环，从queuedRequest中读取Request，然后写入磁盘上的ZKDatabase和log。将处理完的request添加到toFlush中。调用flush()方法一一传递给下一个processor。

  + AckRequestProcessor：直接调用Leader的processAck()方法，模拟发送一个ACK消息给Leader，告知Leader已经落盘完成。其他的Follower也会在落盘完成后发送Leader.ACK消息给Leader，Leader记录落盘完成的节点。当完成的节点数超过Quorum数量的一半后，从outstandingProposals中取出发送的Proposal，检查Proposal中的Request是否已经被认证commit，否则从outstandingProposal中移除Proposal并添加到toBeApplied队列，同时添加Proposal中的request到CommitProcessor（也就是流程的下一个Processor）中的committedRequest队列中，同时发送一个QuorumPacket对象（类型为Leader.COMMIT)给所有的Learner。

- CommitProcessor：注意到从ProposalRequestProcessor传过来的Request是添加到queuedRequest队列中，而Leader的processACK()方法是将认证完成的Request添加到committedProcessor中。这时写请求与读请求不同的地方。同样是循环从queuedRequest中读取Request，读请求可以直接处理，但是写请求需要添加到pendingRequests中，每个session对应更有一个pendingRequest的请求队列。当committedProcessor中有Request时，取出相应session的pendingRequest，对比cxid是否相等，相等的话证明可以认证该Request，因此将pendingRequest交付给下一个Processor执行。注意，这里交付的是pendingRequest，也就是从ProposalRequestProcessor传过来的request，而不是committedRequest中的对象。

- ToBeAppliedRequestProcessor：直接交付给下面的FinalRequestProcessor。写请求的话需要更新toBeApplied队列，表明该写请求已经交付完成。

- FinalRequestProcessor：在ZKDatabase中执行请求，根据Request中的ServerCnxn回复Client。

可以看出，写请求比较复杂，但是可以通过其中的几个关键队列和数据结构来描述写请求的处理流程：Proposal，是指Leader发送给Learner的一个建议，建议所有节点都更新磁盘数据，初始Proposal位于outstandingProposal中，当半数节点落盘完成后，将proposal移交到toBeApplied中表示需要认证交付的写请求；CommitProcessor最为复杂，负责认证Proposal的数据确实落到磁盘了，待认证的Request首先放在queuedRequest中待处理，处理到写请求时添加到pendingRequest中等待认证，当Leader确认半数落盘完成后提交Request到committedRequest中，由CommitProcessor对比两个队列中的当前request是否相同，相同则表示认证完成可以交付。

Leader上的写请求处理流程如下：

![image](http://p1.bqimg.com/4851/f7f57673bef052f1.png)

这其中还有一点需要详细描述的是：Leader发送Proposal给Learner后，Learner如何处理该请求。Learner收到QuorumPacket类型的是Leader.PROPOSAL时，调用FollowerZookeeperServer的logRequest()方法。生成一个新的Request，添加到pendingTxns中，然后调用SyncRequestProcessor处理该Request。因此处理的链是额外的那一条Processor链，如下：

- SyncRequestProcessor：处理写请求的方式同Leader上的SyncRequestProcessor。传入下一个Processor。
- SendAckRequestProcessor：回复一个QuorumPacket对象（类型为Leader.ACK）给Leader。

经过两个Processor的处理，Follower在pendingRequest中维护着刚刚生成的Request，这个Request还需要被认证，因此当它收到Leader发回来的Leader.COMMIT类型的QuorumPacket时，只需要Leader返回的zxid，对比pendingRequest中的zxid，如果相同表示数据是正确的，否则报错，因为Follower上写数据与Leader已经不一致了。移除pendingTxns中的request并交给CommitProcessor处理。从前面的CommitProcessor看出，该Request添加到committedProcessor中，由于这个request的sessionID并没有保存在FollowerZookeeperServer上，所以这个request直接交付给FinalRequestProcessor。FinalRequestProcessor更新完内存数据库后，由于该Request的Cnxn为null，所以无需做response。

Follower上针对Leader上的Proposal的处理必须依赖两个消息Leader.PROPOSAL和Leader.COMMIT，只有它收到这两个消息后才会更新自己的磁盘和内存数据。

#### 2.4.6 Follower上的写请求处理

Follower上的处理和Leader上只有些许不同，流程上有一段是重复的。Request的处理流程如下:

- FollowerRequestProcessor：检查是否是本地请求，是则创建一个虚拟的节点和会话连接。将request添加到LinkedBlockingQueue队列queuedRequest中。线程执行一个死循环，从queuedRequest中取出request，读请求直接传送到下一个processor。但是写请求不同，首先添加到pendingSync队列中，然后Follower调用request()方法发送一个QuorumPacket给Leader，类型为Leader.REQUEST。Leader的LearnerHandler接收到该Learner发送的REQUEST消息后，生产一个新的Request，直接交付给PrepRequestProcessor执行，执行方式和Leader上的一样。

- CommitRequestProcessor：同样，来自于FollowerRequestProcessor的request被添加到queuedRequest中，在循环处理时被添加到pendingRequest中等待认证。当Follower收到Leader.COMMIT消息后，与上文不同的是：这个request会和committedRequest中的request对比一下，然后再提交pendingRequest中的request到FinalRequestProcessor。注意到这里committedRequest中request是一个临时的request，交付的是pendingRequest中的request。

- FinalRequestProcessor：由于CommitRequestProcessor传过来的request有ServerCnxn，所以更新完内存数据库后，需要response。

Follower上的写请求实际上也就是转交给了Leader，在Leader上转交的写请求是同样的处理方式，只是不需要在Leader上response，而是在follower上response。Follower上会在SyncRequestProcessor到CommitRequestProcessor间生成一个临时的Request，用于认证。并且，request的zxid统一由Leader规划指定，Follower上只记住它最后处理的Request的zxid即可。

Follower上的写请求处理流程如下：

![image](http://p1.bqimg.com/4851/5ed2b62c0b48083b.png)
