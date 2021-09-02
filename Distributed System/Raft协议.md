在分布式系统中一致性是很重要的。1990年Leslie Lamport提出基于消息传递的一致性算法 `Paxos` 算法，解决分布式系统中就某个值或决议达成一致的问题。`Paxos` 算法流程繁杂实现起来也比较复杂。

2013年斯坦福的Diego Ongaro、John Ousterhout两个人以**易懂**为目标设计一致性算法Raft。Raft一致性算法保障在任何时候一旦处于leader服务器当掉，可以在剩余正常工作的服务器中选举出新的Leader服务器更新新Leader服务器，修改从服务器的复制目标。



# 多副本状态机 

Replicated State Machine，用来保证容错性。

每台服务器可视为一台状态机，靠日志同步状态。所有服务器按相同顺序执行相同命令，保证各副本最终状态一致。



# Raft一致性算法

`Raft` 将一致性问题分解为了3个子问题：

1. 日志同步：`leader` 在时，由 `leader` 向 `follower` 同步日志
2. `leader` 选举：`leader` 挂掉时，选一个新 `leader`，选举算法
3. 安全性：确保每一个状态机都以相同的顺序执行相同的指令





## Raft协议应用

- redis 选主（采用了类似的做法，但不是直接应用 `Raft`
- 分布式数据库：TiDB
- 服务发现：Consul、etcd





## 和 Paxos 区别

- `Paxos` 侧重于大方向上理论，实现方案各家都不同。`Raft` 作者包含了很多细节，都有伪代码了。
- `Raft` 中，`leader` 必须存有全部 `commited` 的日志。`Paxos` 则不一定，节点即使不包含所有日志也可能选上 `leader`，选上后再靠别的机制补日志。





#### 参考

> [论文原文](https://raft.github.io/raft.pdf)
>
> [论文译文](https://docs.qq.com/doc/DY0VxSkVGWHFYSlZJ)
>
> [牛吃草-Raft协议原理详解](https://zhuanlan.zhihu.com/p/91288179)
>
> [raft算法与paxos算法相比有什么优势，使用场景有什么差异？](https://www.zhihu.com/question/36648084)

