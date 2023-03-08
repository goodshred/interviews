
raft分布式一致性算法

[Redis Cluster去中心化设计的思考](https://heapdump.cn/article/5355727)
## zookeeper
ZAB协议
consul是采用Raft协议来实现的,leader接收到数据,不是立马响应成功,而是把数据同步给follower成功后,才注册成功


较为出名的有etcd，Google的Kubernetes也是用了etcd作为他的服务发现框架

Paxos算法是Leslie Lamport在1990年提出的一种基于消息传递的一致性算法，非常难以理解，基于Paxos协议的数据同步与传统主备方式最大的区别在于：Paxos只需超过半数的副本在线且相互通信正常，就可以保证服务的持续可用，且数据不丢失。

Raft是斯坦福大学的Diego Ongaro、John Ousterhout两个人以易理解为目标设计的一致性算法，已经有了十几种语言的Raft算法实现框架，较为出名的有etcd，Google的Kubernetes也是用了etcd作为他的服务发现框架。

Raft是Paxos的简化版，与Paxos相比，Raft强调的是易理解、易实现，Raft和Paxos一样只要保证超过半数的节点正常就能够提供服务。

ZooKeeper Atomic Broadcast (ZAB, ZooKeeper原子消息广播协议)是ZooKeeper实现分布式数据一致性的核心算法，ZAB借鉴Paxos算法，但又不像Paxos算法那样，是一种通用的分布式一致性算法，它是一种特别为ZooKeeper专门设计的支持崩溃恢复的原子广播协议。


对于没有实际使用、调试、开发过分布式系统的人来说，横向扩展、数据分区、副本/数据分片、容错容灾, 一致性 等和分布式相关的概念、词汇可能会感觉遥远与深奥。


```html
与Zookeeper、raft等实现方式的比较
1. 与使用Zookeeper相比
本篇讲了ES集群中节点相关的几大功能的实现方式：

节点发现
Master选举
错误检测
集群扩缩容
试想下，如果我们使用Zookeeper来实现这几个功能，会带来哪些变化？

Zookeeper介绍
我们首先介绍一下Zookeeper，熟悉的同学可以略过。

Zookeeper分布式服务框架是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。

简单来说，Zookeeper就是用于管理分布式系统中的节点、配置、状态，并完成各个节点间配置和状态的同步等。大量的分布式系统依赖Zookeeper或者是类似的组件。

Zookeeper通过目录树的形式来管理数据，每个节点称为一个znode，每个znode由3部分组成:

stat. 此为状态信息, 描述该znode的版本, 权限等信息.
data. 与该znode关联的数据.
children. 该znode下的子节点.
stat中有一项是ephemeralOwner，如果有值，代表是一个临时节点，临时节点会在session结束后删除，可以用来辅助应用进行master选举和错误检测。

Zookeeper提供watch功能，可以用于监听相应的事件，比如某个znode下的子节点的增减，某个znode本身的增减，某个znode的更新等。

怎么使用Zookeeper实现ES的上述功能
节点发现：每个节点的配置文件中配置一下Zookeeper服务器的地址，节点启动后到Zookeeper中某个目录中注册一个临时的znode。当前集群的master监听这个目录的子节点增减的事件，当发现有新节点时，将新节点加入集群。
master选举：当一个master-eligible node启动时，都尝试到固定位置注册一个名为master的临时znode，如果注册成功，即成为master，如果注册失败则监听这个znode的变化。当master出现故障时，由于是临时znode，会自动删除，这时集群中其他的master-eligible node就会尝试再次注册。使用Zookeeper后其实是把选master变成了抢master。
错误检测：由于节点的znode和master的znode都是临时znode，如果节点故障，会与Zookeeper断开session，znode自动删除。集群的master只需要监听znode变更事件即可，如果master故障，其他的候选master则会监听到master znode被删除的事件，尝试成为新的master。
集群扩缩容：扩缩容将不再需要考虑minimum_master_nodes配置的问题，会变得更容易。
使用Zookeeper的优劣点
使用Zookeeper的好处是，把一些复杂的分布式一致性问题交给Zookeeper来做，ES本身的逻辑就可以简化很多，正确性也有保证，这也是大部分分布式系统实践过的路子。而ES的这套ZenDiscovery机制经历过很多次bug fix，到目前仍有一些边角的场景存在bug，而且运维也不简单。

那为什么ES不使用Zookeeper呢，大概是官方开发觉得增加Zookeeper依赖后会多依赖一个组件，使集群部署变得更复杂，用户在运维时需要多运维一个Zookeeper。

那么在自主实现这条路上，还有什么别的算法选择吗？当然有的，比如raft。

2. 与使用raft相比
raft算法是近几年很火的一个分布式一致性算法，其实现相比paxos简单，在各种分布式系统中也得到了应用。这里不再描述其算法的细节，我们单从master选举算法角度，比较一下raft与ES目前选举算法的异同点：

相同点
多数派原则：必须得到超过半数的选票才能成为master。
选出的leader一定拥有最新已提交数据：在raft中，数据更新的节点不会给数据旧的节点投选票，而当选需要多数派的选票，则当选人一定有最新已提交数据。在es中，version大的节点排序优先级高，同样用于保证这一点。
不同点
正确性论证：raft是一个被论证过正确性的算法，而ES的算法是一个没有经过论证的算法，只能在实践中发现问题，做bug fix，这是我认为最大的不同。
是否有选举周期term：raft引入了选举周期的概念，每轮选举term加1，保证了在同一个term下每个参与人只能投1票。ES在选举时没有term的概念，不能保证每轮每个节点只投一票。
选举的倾向性：raft中只要一个节点拥有最新的已提交的数据，则有机会选举成为master。在ES中，version相同时会按照NodeId排序，总是NodeId小的人优先级高。
看法
raft从正确性上看肯定是更好的选择，而ES的选举算法经过几次bug fix也越来越像raft。当然，在ES最早开发时还没有raft，而未来ES如果继续沿着这个方向走很可能最终就变成一个raft实现。

raft不仅仅是选举，下一篇介绍meta数据一致性时也会继续比较ES目前的实现与raft的异同。
```
# 参考文章
[Elasticsearch分布式一致性原理-与Zookeeper、raft等实现方式的比较](https://zhuanlan.zhihu.com/p/34858035)
