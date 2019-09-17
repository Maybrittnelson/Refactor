* spark的architecture
* 什么是RDD？
* 什么是driver？
* 什么是executor？
* local mode 和 distributed mode的区别
* cluster manager
* parallelism
* 什么是partition？
* 什么是job？
* 什么是stage？
* 什么是task？
* 为什么spark比较快？
* 什么是dataproc?
* 如何create一个spark development environment

[TOC]



### Spark 的 architecture

#### 官网介绍

![Spark cluster components](http://spark.apache.org/docs/latest/img/cluster-overview.png)

**SparkContext** object in your main program（called the Driver program）**coordinated** Spark applications run as independent sets of processes on a cluster.

Several types of **cluster managers**(either Spark's own standalone cluster manager, Mesos or YARN) **allocate ** resources across applications

**Executros** on nodes in the cluster, which are processes that **run** computations and **store** data for your application.

SparkContext sends **tasks** to the excutros to run.

#### 7.2 

**SparkContext：** coordinated、sends tasks（驱动器程序在spark应用中有以下两个职责）

* 为执行器节点coorinated（协调）任务

  Executors（执行器进程） 启动后，会向驱动程序注册自己。因此，执行器进程始终对应用中所有的执行器节点有完整的记录。每个执行器节点，代表能**run** computations (处理任务) 和 **store** data（存储RDD 数据）的进程。

  todo更多调度机制。

* send tasks（把用户程序转为任务）

  Spark 驱动器程序负责把用户程序转换为多个物理执行的单元，这些单元也被称为任务。

  Spark程序其实是隐式地创建出了一个由操作组成的逻辑上的有向无环图（Directed Acyclic Graph，简称DAG）。当驱动程序启动时，它会把这个逻辑图转换成**物理执行单元**。

  