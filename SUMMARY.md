# Saber的后端学习笔记

My study notes


Table of contents:

* [README](README.md)
* Ads
  * [谷歌的广告业务是如何赚钱的](Ads/Google.md)
* Algorithm 算法
  * [KMP算法](Algorithm/KMP算法.md)
  * [Tarjan桥算法](Algorithm/Tarjan桥算法.md)
  * [倍增](Algorithm/倍增.md)
  * [二维凸包 (2D Convex Hull)](Algorithm/凸包.md)
  * [并查集](Algorithm/并查集.md)
  * [最小生成树 (MST)](Algorithm/最小生成树.md)
  * [最短路径](Algorithm/最短路径.md)
  * [环的判断](Algorithm/环的判断.md)
* Career
  * [Career](Career/Career.md)
* Data Structure 数据结构
  * [红黑树](<DataStructure/红黑树.md>)
  * [锁](DataStructure/锁.md)
* Distributed System 分布式
  * [BASE 理论](<DistributedSystem/BASE理论.md>)
  * [CAP](DistributedSystem/CAP.md)
  * [Raft协议](DistributedSystem/Raft协议.md)
  * [Service Mesh](DistributedSystem/服务网格.md)
  * [ZAB协议](DistributedSystem/ZAB协议.md)
  * [分布式事务](DistributedSystem/分布式事务.md)
* Go
  * [Channel的实现](Go/Channel的实现.md)
  * [Context的实现](Go/Context的实现.md)
  * [Go程序性能优化](Go/Go程序性能优化.md)
  * [Go调度-MPG](Go/Go调度-MPG].md)
  * [Go调度-抢占](Go/Go调度-抢占.md)
  * [Go调度-其他](Go/Go调度-其他.md)
  * [Go调度-线程](Go/Go调度-线程.md)
  * [Go调度-译文](Go/Go调度-译文.md)
  * [Timer的实现](Go/Timer的实现.md)
  * [defer的实现](Go/defer的实现.md)
  * [http.Client](Go/http.Client.md)
  * [interface](Go/interface.md)
  * [map的实现](Go/map的实现.md)
  * [mutex的实现](Go/mutex的实现.md)
  * [net.http路由](Go/net.http路由.md)
  * [profile采样的实现](Go/profile采样的实现.md)
  * [slice切片](Go/slice切片.md)
  * [sync.Map的实现](Go/sync.Map的实现.md)
  * [sync.Once的实现](Go/sync.Once的实现.md)
  * [sync.Pool的实现](Go/sync.Pool的实现.md)
  * [sync.Semaphore的实现](Go/sync.Semaphore的实现.md)
  * [内存分配-TCmalloc](Go/内存分配-TCmalloc.md)
  * [内存分配](Go/内存分配.md)
  * [垃圾回收](Go/垃圾回收.md)
  * [坑](Go/坑.md)
  * [Questions](Go/Questions.md)
  * [参考](Go/参考.md)
* Kafka
  * [ISR](Kafka/ISR.md)
  * [架构](kafka/架构.md)
  * [高可用](kafka/高可用.md)
  * [Questions](kafka/Questions.md)
* Network 网络
  * [ARP解析](Network/ARP解析.md)
  * [DNS解析](Network/DNS解析.md)
  * [DPVS](Network/DPVS.md)
  * [GET和POST](Network/GET和POST.md)
  * [HTTP2](Network/Http2.md)
  * [HTTP3](Network/Http3.md)
  * [HTTPS](Network/Https.md)
  * [LVS的转发模式](Network/LVS的转发模式.md)
  * [Nginx](Network/Nginx.md)
  * [OSI七层模型](Network/OSI七层模型.md)
  * [RPC](Network/RPC.md)
  * [RESTFul](Network/RESTFul.md)
  * [socket缓冲区](Network/socket缓冲区.md)
  * [TCP三次握手](Network/TCP三次握手.md)
  * [TCP拥塞控制](Network/TCP拥塞控制.md)
  * [TCP四次挥手](Network/TCP四次挥手.md)
  * [TCP数据结构](Network/TCP数据结构.md)
  * [TCP滑动窗口](Network/TCP滑动窗口.md)
  * [TCP连接四元组](Network/TCP连接四元组.md)
  * [TCP重传机制](Network/TCP重传机制.md)
  * [Questions](Network/Questions.md)
  * [参考](Network/参考.md)
* OS 操作系统
  * [COW写时复制](OS/COW写时复制.md)
  * [IO多路复用](OS/IO多路复用.md)
  * [内存分配](OS/内存分配.md)
  * [内存管理设计精要](OS/内存管理设计精要.md)
  * [虚拟内存](OS/虚拟内存.md)
  * [调度](OS/调度.md)
  * [进程VS线程](OS/进程VS线程g.md)
  * [零拷贝](OS/零拷贝.md)
  * [Questions](OS/Questions.md)
* Redis
  * [Redis6.0多线程](Redis/Redis6.0多线程.md)
  * [Redis分布式锁](Redis/Redis分布式锁.md)
  * [Redis分片](Redis/Redis分片.md)
  * [事务](Redis/事务.md)
  * [参考](Redis/参考.md)
  * [安装](Redis/安装.md)
  * [数据淘汰机制](Redis/数据淘汰机制.md)
  * [数据结构-BloomFilter](Redis/数据结构-BloomFilter.md)
  * [布谷鸟过滤器](Redis/数据结构-CuckooFilter.md)
  * [数据结构-Hashtable](Redis/数据结构-Hashtable.md)
  * [数据结构-Quicklist](Redis/数据结构-Quicklist.md)
  * [SDS](Redis/数据结构-SDS.md)
  * [Set 集合](Redis/数据结构-Set.md)
  * [数据结构-Skiplist](Redis/数据结构-Skiplist.md)
  * [数据结构-Ziplist](Redis/数据结构-Ziplist.md)
  * [数据结构-Zset](Redis/数据结构-Zset.md)
  * [数据结构-hashtable](Redis/数据结构-hashtable.md)
  * [数据结构-quicklist](Redis/数据结构-quicklist.md)
  * [数据结构](Redis/数据结构.md)
  * [缓存一致性](Redis/缓存一致性.md)
  * [通信协议-RESP](Redis/通信协议-RESP.md)
  * [概述](Redis/高可用-Cluster.md)
  * [哨兵模式](Redis/高可用-Sentinel.md)
  * [高可用-主从同步](Redis/高可用-主从同步.md)
  * [持久化](Redis/高可用-持久化.md)
  * [Questions](Redis/Questions.md)
* System Design 系统设计
  * [关注/粉丝列表](<SystemDesign/关注列表.md>)
  * [大文件排序/合并/diff](<SystemDesign/大文件处理.md>)
  * [大纲](<SystemDesign/大纲.md>)
  * [延迟队列](<SystemDesign/延迟队列.md>)
  * [熔断](<SystemDesign/熔断.md>)
  * [秒杀系统](<SystemDesign/秒杀系统.md>)
  * [负载均衡](<SystemDesign/负载均衡.md>)
  * [错误处理](<SystemDesign/错误处理.md>)
  * [限流](<SystemDesign/限流.md>)
  * [Questions](<SystemDesign/Questions.md>)
* DB
  * MySQL
    * [ACID](DB/MySQL/ACID.md)
    * [Log](DB/MySQL/Log.md)
    * [MVCC](DB/MySQL/MVCC.md)
    * [主从同步](DB/MySQL/主从同步.md)
    * [架构](DB/MySQL/架构.md)
    * [索引](DB/MySQL/索引.md)
    * [锁](DB/MySQL/锁.md)
    * [隔离级别](DB/MySQL/隔离级别.md)
    * [页结构](DB/MySQL/页结构.md)
    * [Questions](DB/MySQL/Questions.md)
  * Postgress
    * [Greenpulm](DB/Postgres/Greenpulm.md)
    * [MVCC](DB/Postgres/MVCC.md)
    * [对比MySQL](DB/Postgres/对比MySQL.md)
    * [持久化](DB/Postgres/持久化.md)
    * [隔离级别](DB/Postgres/隔离级别.md)

- Work
  - [bash](Work/bash.md)
  - [code review](Work/CodeReview.md)
  - [iTerm2](Work/iTerm2.md)
  - [mac](Work/mac.md)
  - [postgresql](Work/postgresql.md)
  - [protoc](Work/protoc.md)
  - [ssh](Work/ssh.md)
  - [Tcp相关命令](Work/Tcp相关命令.md)
  - [vim](Work/vim.md)

