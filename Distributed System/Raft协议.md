在分布式系统中一致性是很重要的。1990年Leslie Lamport提出基于消息传递的一致性算法 `Paxos` 算法，解决分布式系统中就某个值或决议达成一致的问题。Paxos算法流程繁杂实现起来也比较复杂。

2013年斯坦福的Diego Ongaro、John Ousterhout两个人以**易懂**为目标设计一致性算法Raft。Raft一致性算法保障在任何时候一旦处于leader服务器当掉，可以在剩余正常工作的服务器中选举出新的Leader服务器更新新Leader服务器，修改从服务器的复制目标。



# 多副本状态机 

Replicated State Machine，用来保证容错性。

每台服务器可视为一台状态机，靠日志同步状态。所有服务器按相同顺序执行相同命令，保证各副本最终状态一致。



# Raft一致性算法

在 `Raft` 体系中，有一个强 `Leader`，由它负责接受客户端的所有命令，将命令作为日志条目复制给其他服务器。在确认安全的时候，将日志命令提交执行（2PC提交）。类似 `MySQL` 的主将 `binlog` 同步给从、或者是 `redis` 的 `AOF` 日志。`Leader` 故障时会触发选举。

`Raft` 将一致性问题分解为了3个子问题：

1. `Leader` 选举
2. 日志复制
3. 安全措施，如确保所有状态机按相同命令执行相同日志。



## Raft协议应用

- redis 主节点选取（采用了类似的做法，但不是直接应用 `Raft`
- 分布式数据库：TiDB
- 服务发现：Consul、etcd



#### 参考

> [论文原文](https://raft.github.io/raft.pdf)
>
> [论文译文](https://docs.qq.com/doc/DY0VxSkVGWHFYSlZJ)
>
> [牛吃草-Raft协议原理详解](https://zhuanlan.zhihu.com/p/91288179)

