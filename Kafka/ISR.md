`kafka` 的 `ISR`（In-Sync Replicas）机制被称为"不丢消息"机制。表示和 `leader` 保持同步的 `follower` 列表



#### kafka 的 Replica

1. `kafka` 的 `topic` 可以设置有N个副本（replica），副本数最好要小于 `broker` 的数量，也就是要保证一个 `broker` 上的 `replica` 最多有一个，这样才能用 `broker id` 指定 `partition`。
2. 创建副本的单位是 `topic` 的分区，每个分区有1个 `leader` 和0到多个 `follower`。





#### kafka 的 "同步"

`kafka` 不是完全同步，也不是完全异步，是一种特殊的 ISR（In Sync Replica）。`ISR` 机制权衡了性能与可用性。

`ISR` 表示和 `leader` 保持同步状态的 `follower` 集合。这里 "同步" 的定义可以配置，默认是 `follower` 落后 `master` 的进度在 **10秒** 以内。

每次 `follower` 来拉数据时， `leader` 会记录各 `follower` 的同步进度，动态维护 `ISR`。`follower` 落后的时长、落后的消息数、`ISR` 的数量都可以配置。





#### 主从同步

<img src="https://pic2.zhimg.com/v2-4d4a6c0ae7218d8fc0659652eab347a5_b.jpg" alt="img" style="zoom:50%;" />

- 共有7条消息，`offset` (消息偏移量) 分别是 0~6
- 0 代表这个日志文件的开始
- `HW (High Watermark)` 为4，0~3 代表这个日志文件可以消费的区间（实际上是 `follower` 已完成同步的偏移量），消费者只能消费到这四条消息
- `LEO` 代表即将要下一个消息可插入的偏移量
- `follower` 拉取数据时，会更新 `leader` 的 `HW`，消费者只能消费 `HW` 前的数据



#### acks参数

- `acks=0`，表示不需要回复，`producer` 不等待 `broker` 的响应，消息会被立即写入缓冲区并视为发送成功。效率最高，但是消息可能会丢。

- `acks=1`，需要一个回复，`producer` 会等待 `leader broker` 收到消息、写入缓冲区、落盘后，才视为发送成功。

- `acks=all/-1`，需要所有节点回复，`leader broker` 收到消息后，挂起，等待 `ISR` 列表中所有 `follower` 返回结果后，才返回。





#### 参考

> [Lovisa Johansson - What does In-Sync Replicas in Apache Kafka Really Mean?](https://www.cloudkarafka.com/blog/what-does-in-sync-in-apache-kafka-really-mean.html)

