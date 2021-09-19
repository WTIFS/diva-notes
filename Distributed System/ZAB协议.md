`ZAB` 全称 `Zookeeper Atomic Broadcast`，Zookeeper 原子消息广播协议。

`ZAB` 是一种专门为 `Zookeeper` 设计的一种支持 **崩溃恢复** 的 **原子广播协议** ，是 `Zookeeper` 保证数据一致性的核心算法。`ZAB` 借鉴了 `Paxos` 算法，但它不是通用的一致性算法，是特别为 `Zookeeper` 设计的。





# 消息传播

`ZAB` 协议的消息广播使用原子广播协议， **类似一个二阶段提交的过程** ，但又有所不同。

1. 第一阶段，`leader` 提一个 `proposal` 给 `follower`。
2. 第二阶段，`leader` 发送 `commit`，通知 `follower` 完成事务的提交。
3. 和一般 `2PC` 不一样，不需要全部 `follower` 回复 `ack`，超过半数 `ack` 就可以才进入第二阶段。





# 主节点选取

## 一些概念

##### election epoch

这是分布式系统中极其重要的概念，由于分布式系统的特点，无法使用精准的时钟来维护事件的先后顺序，因此，Lampert 提出的 `Logical Clock` 就成为了界定事件顺序的最主要方式。

其实就是表示选举的轮数。每次选主，都会加一轮。类似于每个皇帝的年号。



##### zxid

每个消息的编号。

全局唯一。由两部分组成：高32位是 `epoch`，低32位是 `epoch` 内的自增id，由0开始。每次选出新的 `leader`，`epoch` 会递增，同时 `zxid` 的低32位清0。



##### 节点状态

- `LOOKING`: 节点正处于选主状态，不对外提供服务，直至选主结束
- `FOLLOWING`: 作为系统的从节点，接受主节点的更新并写入本地日志
- `LEADING`: 作为系统主节点，接受客户端更新，写入本地日志并复制到从节点





## 选主时机

1. 节点启动时
2. `leader` 节点异常：`leader` 会给 `follower` 发心跳，如果 `follower` 长期未收到 `leader` 的心跳，会进入选主
3. 多数 `follower` 节点异常：如果多数 `follower` 不再响应 `leader` ，那 `leader` 很可能是掉线了，也会进入选主





## 选举条件

1. 选 `epoch` 最大的
2. `epoch` 相等，选 `zxid` 最大的
3. `epoch` 和 `zxid` 都相等，选 `server_id` 最大的（`zoo.cfg` 中配置的 `myid`）

需要过半数节点投票同意，才能选出主节点。





## 选主后的数据同步

这是和 `Paxos` 比较不同的地方。`Paxos` 没有这步。

新的 `leaader` 选出后，可能进度比别的 `follower` 都快，也可能比部分 `follower` 慢。

1. 对 `leader` 上有、`follower` 上没有的数据，`leader` 要重新同步给 `follower`

2. 对 `leader` 上没有、`follower` 上有的，`leader` 要通知 `follower` 丢弃。

通过同步阶段，保证数据的一致性。





# 其他

选主期间，集群不可用



##### 关于一致性的讨论

`ZAB` 的算法是强一致性的，读写都走 `leader`。

但 `zk` 实际应用中，根据配置及使用的接口，是有不同的一致性表现的：

- 它的写是集中在 `leader` 上的，因此写一定是线性一致性的。

- `zk` 默认读走 `follower` 的，可能读到过期数据，这时不是线性的。

- 这种情况下，`zk` 的一致性，介于 **顺序**一致性 和 **线性**一致性之间。(between sequential consistency and linearizability)。
- `zk` 提供了 `sync` 接口，强制 `follower` 从 `leader` 同步数据，保证强一致性，但是会加重 `leader` 负担。



##### 和 Paxos 的异同

`Paxos` 是理论，`ZAB` 是具体实现。两者的实现和概念上有很多相似的地方，比如日志复制、超半数节点同意、纪元的概念等。

两者的**设计目标**也不同，`Paxos` 用于构建一个**复制状态机**，而 `zk` 是用于构建一个**主备系统**。

`Paxos` 下各节点收到的日志后可能是乱序的，但能通过共识算法保证的执行顺序的一致。

而主备系统则要求严格的顺序性保证。



##### zk VS etcd

两者都是通用的一致性元信息存储，也都有 `watch` 来用于变更通知和分发，就这两点，在应用服务的大部分场景下都可以互相替代。

`ZK` 开发的进度和版本更新比较慢了，社区活跃度远远不如 `etcd`。而且应用性上，如果两者都用过的话，应该都会觉得 `etcd` 易用性更高 (restful api)。

其他要考虑周边产品的生态问题了，运维和开发如果是 `Java` 应用更多，那毫无疑问应该用 `ZK`。如果是搞 `Go` 的，那么还是 `etcd` 吧，毕竟有时候遇到问题还是要看源码的。



#### 参考

> [丁凯 - ZAB协议选主过程详解](https://zhuanlan.zhihu.com/p/27335748)
>
> [ZK是强一致性吗?](https://www.zhihu.com/question/455703356)
>
> [ruby - Zab算法与Paxos算法异同](https://www.dazhuanlan.com/rubytoto/topics/1278313)
>
> [runzhliu - zk VS etcd]()(https://www.zhihu.com/question/286230532/answer/773185906)

