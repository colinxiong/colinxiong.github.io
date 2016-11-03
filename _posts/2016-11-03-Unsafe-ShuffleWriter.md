---
published: true
layout: post
title: Unsafe实现Spark中的ShuffleWriter及优化方法
category: spark
tags: 
  - java
  - spark
time: 2016.11.03 22:40:00
excerpt: Spark提供了UnsafeShuffleWriter来解决ShuffleWriter时内存压力大的问题：所有对象全部以bytes形式写入堆外，极大程度上减少JVM堆内的对象。但是这里面有没有优化的点呢？！

---

以下Spark版本为1.4.1。优化部分可以参考Lifetime Based Memory Manager for Distributed Data Processing Systems (PVLDB 2016)。

## 1. Unsafe

Java的sun.misc.Unsafe中定义了Unsafe类，并且是单例模式+私有对象，所以只能通过反射获取Unsafe对象：

```java
// PlatformDependent.java
Field unsafeField = Unsafe.class.getDeclaredField("theUnsafe");
unsafeField.setAccessible(true);
unsafe = (sun.misc.Unsafe) unsafeField.get(null);
```
    
Spark定义了PlatFormDependent来持有静态成员变量UNSAFE，UNSAFE中封装了Unsafe对象。所以不直接调用Unsafe的方法而是通过PlatFormDependent.UNSAFE来调用write和read方法。

---

## 2. ShuffleWriter

Spark的ShuffleWriter就是将计算的RDD重新划分Partition，写入MapOutputTracker和Block，最后写入磁盘做备份。生成写入Block的数据时，对象一直存放在堆中，而且存在大量对象的更迭，所以JVM的内存占用高。UnsafeShuffleWriter则将数据写入堆外，极大减轻内存占用。

重点在于UnsafeShuffleWriter的write方法：

```scala
// ShuffleMapTask.scala
val manager = SparEnv.get.shuffleManager
val writer = manager.getWriter
writer.write(rdd.iterator(partition,context))

// UnsafeShuffleWriter.java
write(scala.collection.Iterator<Product2<K,V>> records)
```

---

## 3. UnsafeShuffleWriter

UnsafeShuffleWriter需要支持Sort、Spill等功能，所以有一些额外的功能类，依次调用为：UnsafeShuffleWriter -> UnsafeShuffleExternalSorter -> PlatformDependent.UNSAFE。

### UnsafeShuffleWriter

- 将RDD写入UnsafeShuffleExternalSorter

逐条写入RDD的每条record。写入一条record的方法是：明确该record所要写入的partition，依次将key和value序列化，然后写入临时的serOutputStream中，再写入sorter。

```java
// UnsafeShuffleWriter.java
void insertRecordIntoSorter(Product2<K, V> record) throws IOException {
    final K key = record._1();
    final int partitionId = partitioner.getPartition(key);
    serBuffer.reset();
    serOutputStream.writeKey(key, OBJECT_CLASS_TAG);
    serOutputStream.writeValue(record._2(), OBJECT_CLASS_TAG);
    serOutputStream.flush();

    final int serializedRecordSize = serBuffer.size();
    assert (serializedRecordSize > 0);

    // Invoke UnsafeShuffleExternalSorter.java
    sorter.insertRecord(
        serBuffer.getBuf(), PlatformDependent.BYTE_ARRAY_OFFSET, serializedRecordSize, partitionId);
    }
```

- 写shuffle文件到磁盘

写入IndexFile，记录当前块在文件中的offset，更新block信息和map task的状态。

```java
// UnsafeShuffleWriter.java
void closeAndWriteOutput() throws IOException {
    serBuffer = null;
    serOutputStream = null;
    final SpillInfo[] spills = sorter.closeAndGetSpills();
    sorter = null;
    final long[] partitionLengths;
    try {
        partitionLengths = mergeSpills(spills);
    } finally {
        for (SpillInfo spill : spills) {
            if (spill.file.exists() && ! spill.file.delete()) {
                logger.error("Error while deleting spill file {}", spill.file.getPath());
            }
        }
    }
    shuffleBlockResolver.writeIndexFile(shuffleId, mapId, partitionLengths);
    mapStatus = MapStatus$.MODULE$.apply(blockManager.shuffleServerId(), partitionLengths);
}
```

### UnsafeShuffleExternalSorter

UnsafeShuffleExternalSorter需要对record进行排序，所以利用了一个指针来指向写入的record。sorter维护一组page，通过currentPage和currentPagePosition指向当前的位置，先写入record的长度，再写入record。写入的Record全部是在堆外！然后写入一组(recordAddress, partitionId)到指针数组。

```java
// UnsafeShuffleExternalSorter.java
final long recordAddress = 
    memoryManager.encodePageNumberAndOffset(currentPage, currentPagePosition);
final Object dataPageBaseObject = currentPage.getBaseObject();
PlatformDependent.UNSAFE.putInt(dataPageBaseObject, currentPagePosition, lengthInBytes);
currentPagePosition += 4;
freeSpaceInCurrentPage -= 4;
PlatformDependent.copyMemory(
    recordBaseObject,
    recordBaseOffset,
    dataPageBaseObject,
    currentPagePosition,
    lengthInBytes);
currentPagePosition += lengthInBytes;
freeSpaceInCurrentPage -= lengthInBytes;
// Invoke UnsafeShuffleInMemorySorter.java
sorter.insertRecord(recordAddress, partitionId);
```

---

## 4. 发现问题、解决问题

通过对比UnsafeShuffleWriter和其他的HashShuffleWriter，SortShuffleWriter，发现一个最主要的问题是：mapSideCombine没有在UnsafeShuffleWriter中应用，因此，UnsafeShuffleWriter会将所有的KV对写入堆外和磁盘。原因是：mapSideCombine参数会导致Map的结果调用aggregator，因此V是可变的，Aggregation操作的V是值的替换，而Non-Aggregation往往是Iterable容器中对象的增加。这对Unsafe是无法支持的，尤其是Non-Aggregation操作，会涉及到V在堆外空间的变化。

最终这个问题导致的结果是：1）写入的KV对较多；2）增加了ShuffleReader的压力；3）而且Unsafe没有ShuffleReader，所以对ShuffleReader来说处理的对象数量非常多。

解决问题的关键是自己实现Unsafe上的aggregator，主要针对的方法是：

```java
// UnsafeShuffleExternalSorter.java
insertRecord()
// new method to insert or update the record when the K has existed.
insertOrUpdateRecord()
```

新方法需要支持的功能：检索当前插入的K是否已经存在，不存在的话插入KV对，存在的话更新V；V的aggregator在Unsafe中的实现。

### K的检索

UnsafeShuffleExternalSorter提供了一个isContain()方法来检验是否存在key，同时还可以获取K在当前Page中的地址。

```java
// UnsafeShuffleExternalSorter.java
private boolean isContain(  Object keyBaseObject,
                            long keyBaseOffset,
                            int keyLengthInBytes,
                            int valueLengthInBytes) {
    final int hashcode = HASHER.hashBytesArray(keyBaseObject,keyBaseOffset, keyLengthInBytes);
    // loc是Location，记录当前处理的Key（如果存在）在Page中的位置
    loc.pos = hashcode & mask;
    int step = 1;
    loc.hashcode = hashcode;

    while (true) {
        if(!bitset.isSet(loc.pos)) {
            // This is a new key.
            loc.setContain(false);
            return false;
        } else {
            long stored = longArray.get(loc.pos * 2 + 1);
            long fullKeyAddress = longArray.get(loc.pos * 2);
            if ((int) (stored) == hashcode) {
                // Full hash code matches.  Let's compare the keys for equality.
                Object page = memoryManager.getPage(fullKeyAddress);
                long position = memoryManager.getOffsetInPage(fullKeyAddress);
                boolean isEqual = true;
                if(keyLengthInBytes +valueLengthInBytes != PlatformDependent.UNSAFE.getInt(page, position)){
                    isEqual =false;
                }
                else{
                    isEqual = ByteArrayMethods.byteAlignedArrayEquals(keyBaseObject,keyBaseOffset,page,position+4,keyLengthInBytes);
                }
                if (isEqual) {
                    loc.setContain(true);
                    loc.setFullKeyAddress(fullKeyAddress);
                    loc.setPage(page);
                    loc.setPosition(position);
                    return true;
                }
            }
        }
        loc.pos = (loc.pos + step) & mask;
        step++;
    }
}
```

### V的aggregator

如果K不存在，那么直接插入KV即可。如果存在K，那么V必然是已经分配了空间了的。所以这个时候如果需要对V做aggregator操作，就需要考虑两点：

 > 1. V的占用空间不变。一般的Aggregation操作，如reduceByKey，是这种但是不保证全是。
 > 2. V的占用空间变化。一般的非Aggregation操作，如groupBykey，是这种，但是不保证全是。

现在问题可以归结为V的空间的问题。

如果V是原生类型，那么我们可以安全的插入和更新V。那除了原生类型还有可能是哪些类型呢？原生类型的组合类型，不可改变长度的数组类型，例如（Int，Long），Array[Int](10)。这些情况都可以安全的插入和更新V。我们称这些类型为静态定长和动态定长类型。

如果V不是上述类型，称为变长类型。变长类型在更新时必然会导致占用空间的变化，这对当前Unsafe的管理是无法控制的。所以，我们只能存一个定长的数据，然后用这个定长的数据来指向一个变长的区域，即地址指针。在更新时，同时更新指针并copy数据即可。变长的区域仍然可以向UNSAFE申请。需要注意的是这种方法下读取V的方法需要相应的做改动。

代码的实现就是接下来一段时间的工作了。^_^