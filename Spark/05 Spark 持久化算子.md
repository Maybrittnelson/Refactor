[TOC]



## Spark 持久化算子

### persist

| 持久化等级                             | 作用                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| MEMORY_ONLY                            | RDD 以反序列化 Java 对象存储在 JVM中。如果RDD在内存中存不下去时，某些 partitions 会不缓存并且每次需要时都重新计算。（这是默认等级） |
| MEMORY_AND_DISK                        | RDD 以反序列化 Java 对象存储在JVM中。如果RDD在内存中存不下去时，会将这些存不下去的分区存储到磁盘上，然后在需要它们时再读取 |
| MEMORY_ONLY_SER（Java and Scala）      | RDD 以序列化 Java 对象存储在JVM中。这通常比反序列化对象更节省空间，特别是在使用[faster serializer](http://spark.apache.org/docs/latest/tuning.html)，但是需要更多的cpu去读 |
| MEMORY_AND_DISK_SER（Java and Scala）  | 同 MEMORY_AND_DISK                                           |
| DISK_ONLY                              | RDD 只存储在磁盘上                                           |
| MEMORY_ONLY_2, MEMORY_AND_DISK_2, etc. | 跟上面的等级一样，不过会复制两份数据到集群中的两个节点       |
| OFF_HEAP (experimental)                | 同 MEMORY_ONLY_SER，但是存储数据到[off-heap memory](http://spark.apache.org/docs/latest/configuration.html#memory-management)。这需要堆外内存被打开。 |

### cache

> 使用的是 persist 的默认等级

### checkpoint

