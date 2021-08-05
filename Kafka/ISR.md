`kafka` 的 `ISR`（In-Sync Replicas）机制被称为"不丢消息"机制。在说 `ISR` 机制前，先讲一下 `kafka` 的副本（replica）。



##### kafka 的 Replica

1. kafka的topic可以设置有N个副本（replica），副本数最好要小于broker的数量，也就是要保证一个broker上的replica最多有一个，所以可以用broker id指定Partition replica。
2. 创建副本的单位是topic的分区，每个分区有1个leader和0到多个follower，我们把多个replica分为 `leader replica` 和 `follower replica`。



**kafka的"同步"**

kafka不是完全同步，也不是完全异步，是一种特殊的 ISR（In Sync Replica）

1. **完全同步复制**要求所有能工作的 follower 都复制完，这条消息才会被认为 commit，这种复制方式极大的影响了吞吐率（高吞吐率是 Kafka 非常重要的一个特性）。
2. **异步复制方式下**，数据只要被 leader 写入 log 就被认为已经 commit，这种情况下如果 leader 宕机，follower 会丢失数据。
3. leader会维持一个与其保持同步的replica集合，该集合就是ISR，每一个partition都有一个ISR，它时有leader动态维护。
4. 我们要保证kafka不丢失message，就要保证ISR这组集合存活（至少有一个存活），并且消息commit成功。



所以我们判定存活的概念是什么呢？分布式消息系统对一个节点是否存活有这样两个条件判断：

1. 节点必须维护和zookeeper的连接，zookeeper通过心跳机制检查每个节点的连接；
2. 如果节点是follower，它必要能及时同步与leader的写操作，不是延时太久。

如果满足上面2个条件，就可以说节点是 "in-sync"（同步中的）。

leader 会追踪 "同步中的" 节点，如果有节点挂了，卡了，或延时太久，那么leader会它移除，延时的时间由参数`replica.log.max.messages` 决定。判断是不是卡住了，由参数 `replica.log.time.max.ms` 决定。



#### acks参数

需要同步完成的 follower 的数量是由 acks 参数来确定的。

1. 当设定为 `1` 的时候仅需要同步给一个 follower 即可
2. 如果为 -1 (all)，则需要同步所有的 follower
3. 如果为 0 的话就代表不需要同步给 follower，记下消息之后立马返回，这样的吞吐量是最好的，但是对消息的也就不能保证丢了
4. 其实常规环境对消息丢失要求没有那么严苛的环境还是可以使用的。常规使用最多的环境应该是设置为 `1`，同步一份就ok了