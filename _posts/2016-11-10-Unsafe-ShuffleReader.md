---
published: true
layout: post
title: Unsafe实现Spark中的ShuffleReader
category: spark
tags: 
  - java
  - scala
  - spark
time: 2016.11.11 00:21:00
excerpt: Spark通过Unsafe实现了ShuffleWriter，尽管存在mapSideCombine的问题，但是已经极大的减少了堆内对象。而Spark没有实现UnsafeShuffleReader，原因是什么？具体能不能用Unsafe实现ShuffleReader呢？

---

以下Spark版本为1.4.1。优化部分可以参考Lifetime Based Memory Manager for Distributed Data Processing Systems (PVLDB 2016)。

前文讲到，ShuffleWriter不支持mapSideCombine的关键一点是V的size在aggregator中不固定。而这也正好是ShuffleReader无法用Unsafe实现的原因。因为ShuffleWriter端可以不支持aggregator，将所有KV对写入Unsafe中；而ShuffleReader必须实现aggregator。

所以，如果要用Unsafe实现ShuffleReader，原理其实和前文提到的K的检索和V的aggregator一样。实际在Spark中有一个约束是，aggregation操作定义了mapSideCombine为true，而non-aggregation操作定义了mapSideCombine为false。这个信息是实现V的aggregator的一个重要信息。另外一个不同点是read是来源于多个不同的地方，需要聚合到一个迭代器中。

## 1. V的size的判断

通过两个简单的例子来分析：

 > reduceByKey(func: (V, V) => V)，将多个V聚合为一个V值，V的长度不变。mapSideCombine为true。

 > groupByKey()，将多个V聚集到一个Iterable的Collection(CompactBuffer)中，V中的元素依次增加，但是size只在array写满后才会增加。mapSideCombine为false。

在我们的初始版本中，我们就先利用mapSideCombine来做一个分类，true表示V的长度不变，而false表示长度会改变。

## 2. ShuffleReader

ShuffleReader首先根据block ID从其他节点获取数据，然后展开成一个迭代器返回给aggregator做操作。做操作时存放的sorter是生命周期最长的对象，所以要将这部分数据存放到UNSAFE中。

从其他节点或者本地获取数据并返回为一个迭代器的方法在BlockStoreShuffleFetcher的fetch方法中：

```scala
def fetch[T](
      shuffleId: Int,
      reduceId: Int,
      context: TaskContext,
      serializer: Serializer)
    : Iterator[T] = {

    // ···

    // 获取数据块的address
    val splitsByAddress = new HashMap[BlockManagerId, ArrayBuffer[(Int, Long)]]
    for (((address, size), index) <- statuses.zipWithIndex) {
        splitsByAddress.getOrElseUpdate(address, ArrayBuffer()) += ((index, size))
    }
    // 获取Block的信息
    val blocksByAddress: Seq[(BlockManagerId, Seq[(BlockId, Long)])] = splitsByAddress.toSeq.map {
        case (address, splits) =>
            (address, splits.map(s => (ShuffleBlockId(shuffleId, s._1, reduceId), s._2)))
    }

    // 根据地址拉取数据的方法
    def unpackBlock(blockPair: (BlockId, Try[Iterator[Any]])) : Iterator[T] = {
        val blockId = blockPair._1
        val blockOption = blockPair._2
        blockOption match {
            case Success(block) => {
                block.asInstanceOf[Iterator[T]]
            }
            case Failure(e) => {
                blockId match {
                    case ShuffleBlockId(shufId, mapId, _) =>
                        val address = statuses(mapId.toInt)._1
                        throw new FetchFailedException(address, shufId.toInt, mapId.toInt, reduceId, e)
                    case _ =>
                        throw new SparkException(
                            "Failed to get block " + blockId + ", which is not a shuffle block", e)
                }
            }
        }
    }
    // 拉取的列表，继承迭代器，所以通过flatMap展开融合
    val blockFetcherItr = new ShuffleBlockFetcherIterator(
        context,
        SparkEnv.get.blockManager.shuffleClient,
        blockManager,
        blocksByAddress,
        serializer,
        // Note: we use getSizeAsMb when no suffix is provided for backwards compatibility
        SparkEnv.get.conf.getSizeAsMb("spark.reducer.maxSizeInFlight", "48m") * 1024 * 1024)
    val itr = blockFetcherItr.flatMap(unpackBlock)

    val completionIter = CompletionIterator[T, Iterator[T]](itr, {
        context.taskMetrics.updateShuffleReadMetrics()
    })

    // 返回的迭代器
    new InterruptibleIterator[T](context, completionIter) {
        val readMetrics = context.taskMetrics.createShuffleReadMetricsForDependency()
        override def next(): T = {
            readMetrics.incRecordsRead(1)
            delegate.next()
        }
    }
}
```

这里返回的迭代器是临时数据！只有在做aggregator的时候，这些对象会一直存活，所以只需要改接下来的aggregator为Unsafe即可。实际上这个rdd就相当于ShuffleWriter里write的参数，后面的写法就和UnsafeShuffleWriter类似了。

---

## 3. UnsafeShuffleReader

将上面产生的rdd(Iterable\[T\])作为参数传入，接下来做的就是aggregator操作和sort操作。完全可以仿照UnsafeShuffleWriter的做法。

### Aggregator操作

aggregator操作分为两种，即前文V的size是否可变。为了实现初步的版本，以K和V均以Int为例来实现。ShuffleHandler中定义了key和value的类型，我们可以获取这些类型，但是这里仅考虑Int。

 > mapSideCombine为true，表示V的size不可变，直接写入KV，调用insertRecordIntoSorter方法。

 > mapSideCombine为false，表示V的size可变，先写入K，再申请一块额外的空间，返回一个地址，写入K后面。申请空间的初始长度为16+4，第一个字节写入存入的V的个数，后面依次写入V。

 做排序时需要按照UnsafeShuffleWriter的方法写一个pointer指针。

 ### 读取操作

 写完后read是返回的一个迭代器，因此需要定义Unsafe上的KV读取方法。

  > hasNext()方法：判断offset是否达到最后一个position

  > next()方法：根据写入的方式读取，如果是mapSideCombine为true，则按序读取K和V（按照ShuffleHandler的类型读取）；如果mapSideCombine为false，则读取K后面的地址后还需要重新读取一遍数据并重组为V。

  这样操作的目的是减少了ShuffleReader阶段aggregator操作部分的大量对象。后续更新实现后的效果吧，也可以参考开头的论文。^_^