- Kafka吞吐量大的原因
  - 磁盘顺序读写
  - 分区分段 + 索引（分topic、broker、segment）
  - 零拷贝（发送使用sendfile，持久化使用mmap）
  - 批量读写 / 压缩

- 如何保证数据能读成功 / 写成功？
  - 生产者：通过ISR（In-Sync Replicas）机制，数据写入主节点和至少1个Follower后才视为成功，否则重试

