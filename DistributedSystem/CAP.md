`CAP` 理论告诉我们，一个分布式系统不可能同时满足以下三种

- 一致性（C: Consistency）
- 可用性（A: Avaibility）
- 分区容错性（P: Partition Tolerance）

这三个基本需求，最多只能同时满足其中的两项，因为P是必须的，因此往往选择就在CP或者AP中。



## Partition Tolerance

所谓分区指的是网络分区的意思。比如你有AB两台服务器，它们之间是有通信的，突然，它们之间的网络链接断开了。那么现在本来AB在同一个网络现在发生了网络分区，变成了A所在的A网络和B所在的B网络。

所谓的分区容忍性，就是说一个数据服务的多台服务器在发生了上述情况的时候，依然能继续提供服务。此时A和C只能二选一。

如果A不写，保持一致性，那么我们就失去了A的服务、A的可用性。但是如果A写了，跟B的数据就不一致了，我们就失去了强一致性。

另一种选择是，我们可以放弃强一致性，选择最终一致性，这也是 `BASE` 理论的做法。比如让A先写入本地，等到通信恢复了再同步给B。长远的看数据还是一致的，只是在某一个时间窗口里数据不一致罢了。



#### 怎么判断 CP/AP

很简单，判断主节点收到客户端请求后，是自己写完就返回（AP），还是同步给从节点后才返回（CP）



#### Redis 保证的是AP

##### 不保证一致性
向 Redis 主节点写入数据后，会立即返回结果。之后在主节点会异步同步给从节点。

主节点挂掉/和从节点无法连接时，从节点仍然是可用的（A），但从节点数据可能和主节点是不一致的（C）。



#### ZooKeeper/Raft 保证的是CP

##### 保证一致性
在基于 `zk` 实现的 `CP` 架构的分布式模型中，向主节点写入数据后，会等待数据的同步结果，当数据在大多数 `zk` 节点间同步成功后，才会返回结果数据。

##### 不保证可用性
不能保证每次服务请求的可用性。如果 `leader` 节点挂了/连不上 `follower`，那么整个集群将暂停对外服务，进入新一轮 `leader` 选举。选举会持续 `30s-120s` 。

`zk` 的 `leader` 每次同步数据给 `follower` 时，也不会确保同步给所有节点，只要有大多数节点（`n/2+1`）同步成功即可。因此如果超过半数节点挂掉，集群会无法选举，也就无法正常服务。



#### 参考

> [facetothefate - CAP 理论常被解释为一种“三选二”定律，这是否是一种误解](https://www.zhihu.com/question/64778723/answer/224266038)
>
> [ZooKeeper和CAP理论及一致性原则](https://blog.csdn.net/yanpenglei/article/details/80362561)