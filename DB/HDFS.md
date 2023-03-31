HDFS，是Hadoop Distributed File System的简称，它是Hadoop的一个子项目，也是Hadoop抽象文件系统的一种实现。HDFS的文件分布在集群机器上，同时提供副本进行容错及可靠性保证。例如客户端写入读取文件的直接操作都是分布在集群各个机器上的，没有单点性能压力。





###  架构设计
#### 服务端

- HDFS的架构设计基于主从复制模式，其中包括一个主节点（NameNode）和多个从节点（DataNode）

- NameNode 负责管理文件系统的命名空间、维护文件系统的元数据信息，也负责确定数据块到具体 Datanode 节点的映射
- DataNode 负责存储实际的数据块，并响应客户端的请求。数据块可以被复制到多个DataNode上，以保证数据的可靠性和高可用性。

#### 客户端

- 客户端会对文件进行切分，分成一个个block进行存储。
- 通过与NameNode进行交互获取文件的位置信息
- 通过与DataNode进行交互读取写入数据
- 客户端本身提供命令用于管理访问HDFS，例如hdfs的开启和关闭



### 高可用

在HDFS中，Namenode可能成为集群的单点故障，Namenode不可用时，整个文件系统是不可用的。HDFS针对单点故障提供了2种解决机制

1. 备份持久化元数据：将文件系统的元数据同时写到多个文件系统， 例如同时将元数据写到本地文件系统及NFS。这些备份操作都是同步的、原子的。

2. 主节点冷备：第二节点定期合并主节点的namespace image和edit log，可用于在主节点完全崩溃时恢复数据



###  数据一致性
为了确保数据的一致性，HDFS采用了主从复制模式。当客户端请求向HDFS写入数据时，先将数据分成多个块，然后将这些块分配到不同的DataNode上进行存储。同时，NameNode会记录每个块存储的位置信息，并将其存储在内存中，以便快速访问。当块的副本发生变化时，NameNode会通知DataNode进行复制或删除。在写入操作完成后，NameNode会将文件的元数据信息存储到磁盘中，以保证数据的一致性。



### 负载均衡

为了提高系统的性能和可扩展性，HDFS采用了多种负载均衡策略，如容量、网络带宽和负载均衡等。其中，容量负载均衡算法会将块分配到剩余磁盘空间最大的DataNode上，以充分利用磁盘空间。网络带宽负载均衡算法会将块分配到距离客户端最近的DataNode上，以最大化网络带宽利用率。负载均衡算法会在DataNode之间移动块，以均衡数据的存储和访问负载，保证系统的性能和可扩展性。



###  容错性

为了确保系统的容错性，HDFS采用了多种机制来处理节点故障。例如，当一个DataNode失效时，NameNode会检测到该节点的失效并将其从系统中删除。同时，为了避免单点故障，HDFS采用了多个NameNode的架构，称为HDFS的主备NameNode机制，确保在一个NameNode节点失效时，备用NameNode可以快速接管其职责，从而保证系统的高可用性。此外，HDFS还支持块复制和数据节点心跳机制来检测数据节点的可用性，以及数据块的复制和重复副本的生成。这些机制的结合，使HDFS在容错性方面表现优异，能够保证数据的持久性和可靠性。