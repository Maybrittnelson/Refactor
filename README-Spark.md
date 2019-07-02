[TOC]

## 调优概述

Spark性能优化的第一步，就是要在开发Spark作业的过程中注意和应用一些性能优化的基本原则。开发调优，就是要让大家了解以下一些Spark基本开发原则，包含：RDD lineage设计、算子的合理使用、特殊操作的优化等。在开发过程中，时时刻刻都应该注意以上原则，并将这些原则根据具体的业务以及实际的应用场景，灵活地运用到自己的Spark作业中。

### 原则一：避免创建重复的RDD

基于重复的数据源创建不同的RDD对象

### 原则二：尽可能复用同一个RDD

### 原则三：对多次使用的RDD进行持久化

### 原则四：尽量避免使用shuffle类算子

shuffle过程：将分布在集中多个节点上同一个key（Q1：存储在哪？），拉取到同一个节点上（网络传输消耗），进行聚合或join等操作（Q2：聚合后存储在哪？，Q3：那为什么要避免使用shuffle 操作？）。

**Q1：**各个节点上的相同key都会先写入本地磁盘文件中，然后其他节点需要通过网络传输拉取各个节点上的磁盘文件中的相同key。

**Q2：**当key过多，导致内存不够存放，进而溢写到磁盘文件中。

**Q3：**shuffle过程中，可能出现大量的磁盘文件读写的IO操作，以及数据网络传输操作。磁盘IO和网络数据传输也是shuffle性能较差的主要原因。

**哪些操作属于shuffle：**reduceByKey、join、distinct、repartition等

```scala
//❌错误示范
val rdd3 = rdd1.join(rdd2)

//✅正确示范,注意：rdd2的数据量较少（比如几百M，或者一两G）的情况下使用。
val rdd2Data = rdd2.collect()
val rrd2DataBroadcast = sc.broadcast(rdd2Data)
val rdd3 = rdd1.map(rdd2DataBroadcast...)
```

### 原则五：使用map-side预聚合的shuffle操作













