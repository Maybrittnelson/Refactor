[TOC]

# Zookeeper

## Zookeeper概念

Zookeeper 是一个分布式协调服务，可用于服务发现，分布式锁，分布式领导选举，配置管理等。

Zookeeper提供了一个类似于Linux文件系统的树形结构（可认为是轻量级的内存文件系统，但只适合存少量信息，完全不适合存储大量文件或者大文件），同时提供了对于每个节点的监控与通知机制。

## Zookeeper 角色

Zookeeper集群是一个基于主从复制的高可用集群，每个服务器承担如下三种角色中的一种

### Leader

1. 一个Zookeeper集群同一时间只会有一个实际工作的Leader，它会发起并维护各Follwer及Observer间的心跳。
2. 所有的写操作必须要通过Leader完成再由Leader将写操作广播给其他服务器。只要有超过半数节点（不包括 observer节点）写入成功，该写请求就会被提交（类2PC协议）。

### Follower

1. 一个Zookeeper集群可能同时存在多个Follower，它会响应Leader的心跳。
2. Follower可直接处理并返回客户端的读请求，同时会将写请求转发给Leader处理。
3. 并且负责在Leader处理写请求对请求进行投票。

### Observer

角色与Follower类似，但是无投票权。Zookeeper需保证高可用和强一致性，为了支持更多的客户端，需要增加更多Server；Server增多，投票阶段延迟增大，影响性能；引入Observer，Observer不参与投票；Observers接受客户端的连接，并将写请求转发给leader节点；加入更多Observer节点，提高伸缩性，同时不影响吞吐率。

### ZAB协议

#### 事务编号Zxid（事务请求计数器+epoch）

在ZAB（Zookeeper Atomic Broadcast， Zookeeper 原子消息广播协议）协议的事务编号Zxid设计中，Zxid是一个64位的数字，其中低32位是一个简单的单调递增的计数器，**针对客户端每一个事务请求，计数器加1**；而高32位则代表Leader周期epoch的编号，**每个当选产生一个新的Leader服务器，就会从这个Leader服务器上取出其本地日志中最大事务的ZXID，并从中读取epoch值，然后加1，以此作为新的epoch**，并将低32位从0开始计数。

Zxid（Transaction id） 类似于RDBMS中的事务ID，用于标识一次更新操作的Proposal（提议）ID。为了保证顺序性，该zxid必须单调递增。

#### epoch

可以理解为当前集群所处的年代或者周期，每个leader就像皇帝，都有自己的年号，所以以每次改朝换代，leader变更之后，都会在前一个年代的基础上加1。这样就算**旧的leader崩溃恢复之后，也没有听他的了，因为follower只听从当前年代的leader的命令**。

#### Zab协议有两种模式-恢复模式（选主）、广播模式（同步）

Zab协议有两种模式，它们分别是**恢复模式（选主）和广播模式（同步）**。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和leader的状态同步以后，恢复模式就结束了。状态同步了leader和Server具有同步的系统状态。

#### Zab 协议 4阶段

#### Leader election（选举阶段-选出准Leader）

Leader election（选举阶段）：节点在一开始都处于选举阶段，只要有一个节点得到超半数节点的票数，它就可以当选准leader。只有到达广播阶段（broadcast）准leader才会成为真正的leader。这一阶段的目的就是为了选出一个**准leader**，然后进入下一个阶段。

#### Discovery（发现阶段-接受提议、生成epoch、接受epoch）

Discovery（发现阶段）：在这个阶段，**followers 跟准 leader 进行通信，同步 followers 最近接收的事务提议。** 这个一阶段的**主要目的是发现当前大多数节点接收的最新提议**，并且**准leader生成新的epoch，让followers接受，更新它们的accepted Epoch**。

一个follower只会连接一个leader，**如果有一个节点f认为另一个follower p 是 leader， f 在尝试连接 p 时会被拒绝，f被拒绝之后，就会进入重新选举阶段**。

#### Synchronization（同步阶段-同步follower副本）

Synchronization（同步阶段）：同步阶段主要是利用**leader前一阶段获得的最新提议历史，同步集群中所有的副本。只有当大多数节点都同步完成，准 leader 才会成为真正的 leader**。

follower只会接受zxid比自己的lastZxid大的提议。

#### Broadcast（广播阶段-leader消息广播）

Broadcast（广播阶段）：到了这个阶段，Zookeeper集群才能正式对外提供事务服务，并且leader可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。

ZAB 提交事务并不像2PC一样需要全部follower 都 ACK，**只需要得到超过半数的节点的ACK就可以了**。

#### ZAB协议JAVA实现（FLE-发现阶段和同步合并为Recovery Phase（恢复阶段））

协议的Java 版本实现跟上面的定义有些不同，选举阶段使用的是Fast Leader Election（FLE），它包含了选举的发现职责。因为FLE 会选举拥有最新提议历史的节点作为leader，这样就省去了发现最新提议的步骤。实际的实现将 发现阶段 和 同步合并为 Recovery Phase（恢复阶段）。所以 ，ZAB的实现有三个阶段：Fast Leader Election；Recovery Phase； Broadcast Phase。

### 投票机制

**每个server首先给自己投票，然后用自己的选票和其他server选票对比，权重大的胜出，使用权重较大的更新自身选票箱**。具体选举过程如下：

1. 每个 Server 启动以后**都询问其它的Server它要投票给谁**。对于其他server的询问，server每次根据自己的状态都回复自己推荐的leader的id和上一次处理事务的zxid（系统启动时每个server都会推荐自己）
2. 收到所有 Server 回复以后，**就计算出 zxid 最大的哪个Server**，并将这个Server相关信息设置成下一个要投票的Server
3. 计算这个过程中**获得票数最多的 server 为获胜者**，如果获胜者的票数超过半数，则该 server 被选为leader。否则，继续这个过程，直到leader被选举出来
4. leader 就会开始等待 server 连接
5. Follower 连接 leader，将最大的 zxid 发送给 leader
6. Leader 根据 follower 的 zxid 确定同步点，至此选举阶段完成
7. 选举阶段完成 Leader 同步后通知 follower 已经成为 uptodate 状态
8. Follower 收到 uptodate 消息后，又可以重新接受 client 的请求进行服务了

目前有 5 台服务器，每台服务器均没有数据，它们的编号分别是1，2，3，4，5按编号依次启动，它们的选举过程如下：

1. 服务器 1 启动，给自己投票，然后发投票信息，由于其他机器还没有启动所以它收不到反馈信息，服务器1的状态一直属于 Looking。
2. 服务器 2 启动，给自己投票，同时与之前启动的服务器1交换结果，由于服务器2的编号大所以服务2胜出，但此时投票数没有大于半数，所以两个服务器的状态是 LOOKING。
3. 服务器 3 启动，给自己投票，同时与之前启动的服务器1，2交换信息，由于服务器 3 的编号最后所以服务器 3 胜出，此时投票数正好大于半数，所以服务器 3 成为领导者，服务器 1，2 成为小弟。
4. 服务器 4 启动， 给自己投票，同时与之前启动的服务器1，2，3交换信息，尽管服务器4的编号大，但之前服务器3已经胜出，所以服务器4知恩感成为小弟。
5. 服务器 5 启动， 后面的逻辑同服务器 4 成为小弟。

## Zookeeper工作原理（原理广播）

1. Zookeeper的核心是原子广播，这个机制保证了各个server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是恢复模式和广播模式
2. 当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数 server 的完成了和 leader 的状态同步以后，恢复模式就结束了。
3. 状态同步保证了 leader 和 server 具有相同的系统状态
4. **一旦 leader 已经和多数的 follower 进行了状态同步后，他就可以开始广播消息了**，即进入广播消息。这时候当一个 server 加入 zookeeper 服务中，它会在恢复模式下启动，发现 leader， 并和 leader 进行状态同步。待到同步结束，它也参与消息广播。Zookeeper服务一致维护在 Broadcast 状态，直到 leader 崩溃了或者 leader 失去了大部分的 followers 支持。
5. 广播模式需要保证 proposal 被按顺序处理，因此 zk 采用了递增的事务id号（zxid）来保证。所有的提议（proposal）都在被提出的时候加上了 zxid。
6. 实现中 zxid 是一个64位的数字，它高 32 位是 epoch 用来标识 leader 关系是否改变，每次一个 leader 被选举出来，它都会有一个新的 epoch。低 32 位是递增计数。
7. 当 leader 崩溃或者 leader 失去大多数的 follower， 这时候 zk 进入恢复模式，恢复模式需要重新选举出一个新的 leader，让所有的 server 都恢复到一个正确的状态

### Znode 有四种形式的目录节点

1. PERSISTENT：持久的节点
2. EPHEMERAL：暂时的几点
3. PERSISTENT_SEQUENTIAL：持久化顺序编号目录节点
4. EPHEMERAL_SEQUENTIAL：暂时化顺序编号目录节点

