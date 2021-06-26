> 想必大部分同学跟我一样对`Zookeeper`的八股文特性一头雾水，到底什么叫分布式协调？什么命名发现？什么是`Watcher`？虽然每个字都认识，但就是......于是拿来`Kafka`, `HBase`和`Kubernetes`一众分布式系统来探究到底分布式系统为什么要用`Zookeeper`，`Etcd`这样一些*分布式协调系统*，它们之间是如何交互的。

### CAP中的P

<img src="https://secureservercdn.net/166.62.107.20/lml.0a0.myftpupload.com/wp-content/uploads/2020/06/CAP-Partition-Tolerance.png" alt="CAP" style="zoom:50%;" />

> 推荐阅读[CAP理论中的P到底是个什么意思？](https://www.zhihu.com/question/54105974/answer/1643846752)

分布式系统中有`Node1` - `Node5`这样5个节点，正常情况下它们各自对外提供服务，同时又有一些例如配置**共享的数据**在各个节点中存储，其中某个节点在更新这些数据后需要通知并让其他节点同步此修改，确保集群中所有节点的数据都是完全一致的，这就是对数据**Consistency**要求。

这时因为宕机或者网络故障等原因，其中某些节点(N4, N5或者说N1, N2, N3)与整个网络中的其他节点进行通信和数据同步，这是后就形成**P网络分区**(Network **Partition**)。这时候就需要集群来选择到底是要**鱼 - Availability**还是**熊掌 - Consistency**了。

如果选择**鱼 - Availability**，那么集群中的每个节点持续对外提供服务，但不保证共享数据的一致性；如果选择**熊掌 - Consistency**，那么必须让集群一部分网络分区的节点停止对外的服务从而保证数据的一致性。

> 既然网络通信故障了，怎么控制哪些节点停止对外服务？
>
> 如果这些分区后的节点恢复正常后，怎么保证这些节点的数据和另外分区节点的数据一致？

### 分布式系统的问题

这些问题就是靠`Paxos`和[Raft](http://thesecretlivesofdata.com/raft/)这样一些分布式共识算法来解决，像`Zookeeper`, `etcd`这样一些分布式协调系统就会实现一套完整的解决方案来将分布式系统的共性问题解决了，让`Kafka`，`Kubernetes`这些系统专注于分布式业务本身。

怎么来理解这句八股文呢？

首先自己实现`Paxos`或者`Raft`算法的复杂度很高，你需要实现：

- 集群中的**Leader 选举**，确保同一时刻对外只有一个Leader于Client沟通
- 通过**Log Replication**机制同步Leader和其他节点之间数据，包括集群中的节点角色发生变化时，其中某些节点退出或者加入集群等情况
- 在集群中的节点，服务状态发生变化是，要及时**通知Client**感知此变更

而这些特性的实现又要考虑其**强一致性**，**高可靠性**，**高可用性**和**高性能**，因此通过`Zookeeper`, `etcd`提供API来支持分布式一致性的实现来帮助`Kafka`，`Kubernetes`这些分布式系统来实现可靠的分布式服务。

### Zookeeper etcd提供的能力

在分布式系统中参与者有三部分：

**Client**

> 例如Kafka中的Consumer和Producer

Client注册和注销，以及**分布式服务**中角色和状态发生变化时Client需要及时感知

**分布式协调者**

> Zookeeper和etcd

**分布式服务**

> Kafka和Kubernetes

分布式服务进行Leader选举，保证Client只和服务集群中的Leader交互，保持Leader和Follower之间的心跳。

#### 高可用配置数据库

> 配置管理

**What? 数据库？**没错，正如`etcd`的原本意思，它就是Linux `etc`目录和`distributed`的意思，即包括配置的地方。只不过`etcd`是以KV的形式，`Zookeeper`是以文件系的形式。

在Kubernetes中集群的配置，状态，pod的数量及状态，Kubernetes API相关对象等都会存储在etcd中。而在Kafka集群中topic 与 broker 的对应关系，消费者注册信息，Topic消费 Offset等也是存储在Zookeeper中。

```
/consumers
/brokers/seqid
/brokers/ids
/brokers/topics
/config
/config/topics
/config/client
/admin/delete_topics
```

> 那能不能用MySQL或者Redis呢？

能，至少对**配置管理**这块我们完全可以用外部存储来实现，只需要保证存储服务的高可用，强一致性，高可靠，高性能。`etcd`的WAL，`Zookeeper`的事务日志和快照跟数据库原理一样，都是实现一致性数据存储。只不过`etcd`和`Zookeeper`通常都有多个节点提供数据读写服务，在不同节点之间进行日志复制和提交来实现**高可用**。

<img src="https://tech.ebayinc.com/assets/Uploads/Editor/Raft2.png" alt="Log Replication" style="zoom:50%;" />



### Leader选举

可能所有的文章都讲了如何进行Leader选举，但对Leader在集群里面的作用没有讲的很清楚。**此处的Leader是`kafka`和`Kubernetes`分布式服务集群中的Leader，而不是分布式协调服务，分布式服务集群通过分布式协调服务提供的服务进行Leader选举。**

`Leader`在集群中负责：

**与Client的交互**

在`Kafka`中每个`Topic`的`Partition`在多个`Worker`上都会选举出来一个`Leader`，只有`Leader`处理`Consumer`和`Producer`的读写请求

<img src="http://cloudurable.com/images/kafka-architecture-topics-replication-to-partition-0.png" style="zoom:70%;" />

**负责Leader于Follower的数据同步**

`Leader`在接收到数据写入请求后负责数据在其他`Follower`节点上的同步，提交数据变更日志后返回客户端

**负责Leader和Follower的心跳**

维护`Leader`和`Follower`之间的心跳

### Watcher机制

相对`Zookeeper`集群来说，`Kafka`的`Broker`和`Producer`, `Consumer`都是Client，当存储在`Zookeeper`中的目录发生变化是都会通知注册了事件的客户端，这样一来当`Broker`挂掉或者重新选举产生新的Leader，客户端都可以感知此变化。

在`Kubernetes`集群中，当连接`apiserver`的客户端或者`kubectl`客户端注册监听后，当Watch对象的状态发生变化时也会通知客户端。典型的例子就是`kubectl get pods -n my-namespace --watch`，在加入`--watch`选项后，当`apiserver`中pod的状态发生变化时都会通知`kubectl`客户端。

### 参考

[NuRaft: a Lightweight C++ Raft Core](https://tech.ebayinc.com/engineering/nuraft-a-lightweight-c-raft-core/)

[深入浅出理解基于 Kafka 和 ZooKeeper 的分布式消息队列](https://gitbook.cn/books/5ae1e77197c22f130e67ec4e/index.html)

[Kafka Topic Architecture - Replication, Failover and Parallel Processing](http://cloudurable.com/blog/kafka-architecture-topics/index.html)

[etcd & Kubernetes: What You Should Know](https://rafay.co/the-kubernetes-current/etcd-kubernetes-what-you-should-know/)
