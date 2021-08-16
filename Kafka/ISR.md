`kafka` 的 `ISR`（In-Sync Replicas）机制被称为"不丢消息"机制。在说 `ISR` 机制前，先讲一下 `kafka` 的副本（replica）。



#### kafka 的 Replica

1. kafka 的 `topic` 可以设置有N个副本（replica），副本数最好要小于 `broker` 的数量，也就是要保证一个 `broker` 上的 `replica` 最多有一个，这样才能用 `broker id` 指定 `partition`。
2. 创建副本的单位是 `topic` 的分区，每个分区有1个 `leader` 和0到多个 `follower`。



#### kafka 的 "同步"

`kafka` 不是完全同步，也不是完全异步，是一种特殊的 ISR（In Sync Replica）。`ISR` 机制权衡了性能与可用性。

`ISR` 表示和 `leader` 保持同步状态的 `follower` 集合。这里 "同步" 的定义可以配置，默认是 `follower` 落后 `master` 的进度在 10秒 以内。

每次 `follower` 来拉数据时， `leader` 会记录各 `follower` 的同步进度，动态维护 `ISR`。`follower` 落后的时长、落后的消息数、`ISR` 的数量都可以配置。



#### acks参数

- `acks=0`，`producer` 不等待 `broker` 的响应，消息会被立即写入缓冲区并视为发送成功。效率最高，但是消息可能会丢。

- `acks=1`，`producer` 会等待 `leader broker` 收到消息、写入缓冲区、落盘后，才视为发送成功。

- `acks=all/-1`，`leader broker` 收到消息后，挂起，等待所有 `ISR` 列表中的 `follower` 返回结果后，再返回 `ack`。





#### 参考

> [Lovisa Johansson - What does In-Sync Replicas in Apache Kafka Really Mean?](https://www.cloudkarafka.com/blog/what-does-in-sync-in-apache-kafka-really-mean.html)

