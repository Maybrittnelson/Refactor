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

如果因为业务需要，一定要使用shuffle操作，无法用map类的算子来替代，那么尽量使用可以map-side预聚合的算子。

所谓的map-side预聚合，说的是在每个节点本地对相同的key进行一次聚合操作，类似于MapReduce中的本地combiner。map-side预聚合之后，每个节点本地就只会有一条相同的key，因为多条相同的key都被聚合起来了。其他节点在拉取所有节点上的相同key时，就会大大减少需要拉取的数据数量，从而也就减少了磁盘IO以及网络传输开销。通常来说，在可能的情况下，建议使用reduceByKey或者aggregateByKey算子来代替groupByKey算子。因为reduceByKey和aggregateByKey算子都会使用用户自定义的函数对每个节点本地的相同key进行预聚合。而groupByKey算子时不会进行预聚合的，全量的数据会在集群的各个节点之间分发和传输，性能相对来说比较差。

比如如下两幅图，就是典型的例子，分别基于reduceByKey和groupByKey进行单词计数。其中第一张图是groupByKey的原理图，可以看到，没有进行任何本地聚合时，所有数据都会在集群节点之间传输；第二张图是reduceByKey的原理图，可以看到，每个节点本地的相同key数据，都进行了预聚合，然后才传输到其他节点上进行全局聚合。

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/5ebe0848.png)

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/a6c7d4c4.png)

### 原则六：使用高性能的算子

除了shuffle相关的算子有优化原则之外，其他的算子也都有着对应的优化原则。

#### 使用reduceByKey/aggregateByKey替代groupByKey

#### 使用mapPartitions替代普通map

mapPartitions类的算子，一次函数调用会处理一个partition所有的数据，而不是一次函数调用处理一条，性能相对来说会高一些。但是有时候，使用mapPartitions会出现OOM（内存溢出）的问题。因为单次函数调用就要处理掉一个partition所有的数据，如果内存不够，垃圾回收时是无法回收掉太多对象的，很可能出现OOM异常。所有使用这类操作时要慎重！

#### 使用foreachPartitions替代foreach

原理类似于"使用mapPartitions替代map"，也是一次函数调用处理一个partition的所有数据，而不是一次函数调用处理一条数据。在实践中发现，foreachPartitions类的算子，对性能的提升还是很有帮助的。比如在foreach函数中，将RDD中所有数据写MySQL，那么如果是普通的foreach算子，就会一条数据一条数据地写，每次函数调用可能就会创建一个数据库连接，此时就势必会频繁地创建和销毁数据库连接，性能是非常低下；但是如果用foreachPartitions算子一次性处理一个partition的数据，那么对于每个partition，只要创建一个数据库连接即可，然后执行批量插入操作，此时性能是比较高的。实践中发现，对于1万条左右的数据量写入MySQL，性能可以提升30%以上。

#### 使用filter之后进行coalesce操作

通常对一个RDD执行filter算子过滤掉RDD中较多数据后（比如30%以上的数据），建议使用coalesce算子，手动减少RDD的partition数量，将RDD中的数据压缩到更少的partition中去。因为filter之后，RDD的每个partition中都会有很多数据被过滤掉，此时如果照常进行后续的计算，其实每个task处理的partition中的数据量并不是很多，有一点资源浪费，而且此时处理的task越多，可能速度反而越慢。因此用coalesce减少partition数量，将RDD中的数据压缩到更少的partition之后，只要使用更少的task即可处理完所有的partition。在某些场景下，对于性能的提升会有一定的帮助。

#### 使用repartitionAndSortWithinPartitions替代repartition与sort类操作

repartitionAndSortWithinPartitions是Spark官网推荐的一个算子，官方建议，如果需要在repartition重分区之后，还要进行排序，建议直接使用repartitionAndSortWithinPartitions算子。因为该算子可以一边进行重分区的shuffle操作，一边进行排序。shuffle与sort两个操作同时进行，比先shuffle再sort来说，性能可能要高的。

### 原则七：广播大变量

有时在开发过程中，会遇到需要在算子函数中使用外部变量的场景（尤其是大变量，比如100M以上的大集合），那么此时就应该使用Spark的广播（Broadcast）功能来提升性能。

在算子函数中使用到外部变量时，默认情况下，Spark会将该变量复制多个副本，通过网络传输task中，此时每个task都有一个变量副本。如果变量本身比较大的话（比如100M，甚至1G），那么大量的变量副本在网络中传输的性能开销，以及在各个节点的Executor中占用过多内存导致的频繁GC，都会极大地影响性能。

因此对于上述情况，如果使用的外部变量比较大，建议使用Spark的广播功能，对该变量进行广播。广播后的变量，会保证每个Executor的内存中，只驻留一份变量副本，而Executor中的task执行时共享该Executor中的那份变量副本。这样的话，可以大大减少变量副本的数量，从而减少网络传输的性能开销，并减少对Executor内存的占用开销，降低GC的频率。

### 原则八：使用Kryo优化序列化性能

在Spark中，主要有三个地方涉及到了序列化：

* 在算子函数中使用到外部变量时，该变量会被序列化后进行网络传输（见“原则七：广播大变量”中的讲解）
* 将自定义的类型作为RDD的范型类时（比如JavaRDD，Student是自定义类型），所有自定义类型对象，都会进行序列话。因为这种的情况下，也要求自定义的类必须实现Serializable接口
* 使用可序列化的持久策略时（比如MEMORY_ONLY_SER），Spark会将RDD的每个partition都序列化成一个大的字节数组。

对于这三种出现序列化的地方，我们可以通过使用Kryo序列化类库，来优化序列化和反序列化的性能。Spark默认使用的是Java的序列化机制，也就是ObjectOutputStream/ObjectInputStream API来进行序列话和反序列化。但是Spark同时支持使用Kryo序列化库，Kryo序列化类库的性能比Java序列化类库的性能要高很多。官方介绍，Kryo序列化机制比Java序列化机制，性能高10倍左右。Spark之所以默认没有使用Kryo作为序列化类库，是因为Kryo要求最好要注册所有需要进行序列化的自定义类型，因此对于开发者来说，这种方式比较麻烦。

以下是使用Kryo的代码示例，我们只要设置序列化类，再注册要序列化的自定义类型即可（比如算子函数中使用到的外部变量类型、作为RDD泛型类型的自定义类型等）：

```scala
//创建SparkConf对象。
val conf = new SparkConf().setMaster().setAppName()
// 设置序列化器为KryoSerializer
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
// 注册要序列化的自定义类型
conf.registerKryoClasses(Array(classOf[MyClass1], classof[MyClass2]))
```

### 原则九：优化数据结构

Java中，有三种类型比较耗费内存：

* 对象，每个Java对象都有对象头、引用等额外的信息，因此比较占用内存空间。
* 字符串，每个字符串内部都有一个字符串数组以及长度等额外信息。
* 集合类型，比如HashMap、LinkedList等，因为集合类型内部通常会使用一些内部类来封装集合元素，比如Map.Entry。

因此Spark官方建议，在Spark编码实现中，特别是对于算子函数中的代码，尽量不要使用上述三种数据结构，尽量使用字符串代替对象，使用原始类型（比如Int、Long）替代字符串，使用数组替代集合类型，这样尽可能地减少内存占用，从而降低GC频率，提升性能。

但是在笔者的编码实践中发现，要做到该原则其实并不容易。因为我们同时考虑到代码的可维护性，如果一个代码中，完全没有任何对象抽象，全部是字符串拼接的方式，那么对于后续的代码维护和修改，无疑是一场巨大灾难。同理，如果所有操作都是基于数组实现，而不使用HashMap、LinkedList等集合类型，那么对于我们的编码难度以及代码可维护性，也是一个极大的挑战。

## 参数调优

### Spark作业基本运行原理

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2016/1f1ddad5.png)

spark-submit提交一个Spark作业之后，这个作业就会启动一个对应的Driver进程。根据你使用的部署（deploy-mode）不同，Driver进程可能在本地启动，也可能在集群中某个工作节点上启动。Driver进程本身会根据我们设置的参数，占有一定数量的内存和CPU core。

1. Dirver进程向集群管理器申请运行Spark作业需要使用的资源，这里的资源指的就是Executor进程。
2. Yarn集群管理器会根据我们为Spark作业设置的资源参数，在各个工作节点上，启动一定数量的Executor进程，每个Executor进程都占有一定数量的内存和CPU core。
3. 在申请到作业执行所需的资源之后，Driver进程就会开始调度和执行我们编写的作业代码了。
4. Driver进程会将我们编写的Spark作业代码分拆为多个stage，每个stage执行一部分代码片段，并为每个stage创建一批task，然后将这些task分配到各个Executor进程中执行。
5. task是最小的计算单元，负责执行一摸一样的计算逻辑，只是每个task处理的数据不同而已。
6. 一个stage的所有task都执行完毕之后，会在各个节点本地的磁盘文件中写入计算中间结果，然后Driver就会调度运行下一个stage。
7. 下一个stage的task的输入数据就是上一个stage输出的中间结果。如此循环往复，直到将我们自己编写的代码逻辑全部执行完，并且计算完所有的数据，得到我们想到的结果为止。



#### 补充

* Spark是根据shuffle类算子来进行stage的划分。如果我们的代码中执行了某个shuffle类算子（比如reduceByKey、join等），那么就会在该算子处，划分出一个stage界限来。可以大致理解为，shuffle算子执行之前的代码会被划分为一个stage，shuffle算子执行以及之后的代码会被划分为下一个stage。因此一个stage刚开始执行的时候，它的每个task可能都会从上一个stage的task所在的节点，去通过网络传输拉取需要自己处理的所有key，然后对拉取的所有相同key使用我们自己编写的算子函数执行聚合操作（比如reduceByKey算子接受的函数）。**这个过程就是shuffle**。
* 当我们在代码执行了cache/persist等持久化操作时，根据我们选择的持久化级别不同，每个task计算出来的数据会保存到Executor进程的内存或者所在节点的磁盘文件中。因此Executor的内存主要分为三块：第一块是让task执行我们自己编写的代码时使用，默认是占Executor占总内存的20%；第二块是让task通过shuffle过程拉取上一个stage的task的输出后，进行聚合等操作时使用，默认也是占Executor总内存的20%；第三块是让RDD持久化时使用，默认Executor总内存的60%。
* task的执行速度是每个Executor进程的CPU core数量有直接关系的。一个CPU core同一时间只能执行一个线程。而每个Executor进程上分配的多个task，都会以每个task一条线程的方式，多线程并发运行的。如果CPU core数量比较充足，而且分配到的task数量比较合理，可以比较快速和高效地执行这些task线程。

### 具体参数

#### num-executors

* 参数说明：该参数用于设置Spark作业总共要用多少个Executor进程来执行。Driver在向YARN集群管理器申请资源时，YARN集群管理器会尽可能按照你的设置来集群的各个工作节点上，启动相应数量的Executor进行。这个参数非常重要，如果不设置的话，默认只会给你启动少量的Executor进程，此时你的Spark作业的运行速度是非常慢的。
* 参数调优建议：50～100

#### executor-memory

* 参数说明：该参数用于设置每个Executor进程的内存。Executor内存的大小，很多时候决定了Spark作业的性能，而且跟常见的JVM OOM异常，也有直接的关联。
* 参数调优建议：每个Executor进程的内存设置4G-8G较为合适。但是这只是一个参考值，具体的设置还得根据不同部门的资源队列来定。可以看看自己团队的资源队列的最大内存限制是多少，num-executors乘以executor-memory，是不能超过列的最大内存量的。此外，如果你是跟团队里其他人共享这个资源队列，那么申请的内存量最好不要超过资源队列最大总内存的1/3～1/2，避免你自己的Spark作业占用了队列所有的资源，导致别的同学的作业无法运行。

#### executor-cores

* 参数说明：该参数用于设置每个Executor进程的CPU core数量。这个参数决定了每个Executor进程并行执行task线程的能力。因为每个CPU core同一时间只能执行一个task线程，因此每个Executor进程CPU core数量越多，越能够快速地执行完分配给自己的所有task线程。
* 参数调优建议：如果跟他人分享这个队列，那么num-executors * executor-cores不要超过队列总CPU core的1/3～1/2左右比较合适，也是避免影响其他同学的作业运行。

#### driver-memory

* 参数说明： 该参数用设置Driver进程的内存。
* 参数调优建议：Driver的内存通常来说不设置，或者设置1G左右应该就够了。唯一需要注意的一点是，如果需要使用collect算子将RDD的数据拉取到Driver上进行处理，那么必须保Drvier的内存足够大，否则会出现OOM内存溢出的问题。

#### spark.default.parallelism

* 参数说明：该参数用于设置每个stage的默认task数量。这个参数极为重要，如果不设置可能会直接影响你的spark作业性能。
* 参数调优建议：num-excutors*excutor-cores的2～3较为合适，那么设置1000个task是可以的

#### spark.storage.memoryFraction

* 参数说明：该参数用于设置RDD持久化数据在Executor内存中能占的比例，默认是0.6。也就说，默认Executor60%的内存，可以用来保存持久化的RDD数据。根据你选择的不同的持久化策略，如果内存不够时，可能数据不会持久化，或者数据会写入磁盘。
* 参数调优建议：如果Spark作业中，有较多的RDD持久化操作，该参数的值可以适当提高一些，保证持久化的数据能够容纳在内存中。避免内存不够缓存所有的数据，导致数据只能写入磁盘中，降低了性能。但是如果Spark作业中的shuffle类操作比较多，而持久化操作比较少，那么这个参数的值适当降低一些比较合适。此外，如果发现作业由于频繁的gc导致运行缓慢（通过spark web ui 可以观察作业的gc耗时），意味着task执行用户代码的内存不够用，那么同样建议降低这个参数的值。





