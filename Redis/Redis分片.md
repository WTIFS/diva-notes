### **分布式寻址算法**

- 客户端分片
  - `hash` 算法（扩容时会导致大量缓存重建）
  - 一致性 `hash` 算法
- 服务端分片
  - `redis cluster` 的 `hash slot` 算法





### hash 算法

来了一个 `key`，首先计算 `hash` 值，然后对节点数取模。然后打在不同的节点上。

但是当服务器数目发送增加或减少时，分片方式会变为 `key%(N+1)` 或 `key%(N-1)`，这会导致大量旧节点中的 `key` 无法命中新节点





### 一致性 hash 算法

一致性 `hash` 算法将整个 `hash` 值空间 (`32`位哈希就是 `0 ~ 2^32`) 组织成一个虚拟的圆环，整个空间按顺时针方向组织，下一步将各个节点（如服务器的 `ip` 或主机名）进行 `hash`。这样就能确定每个节点在其哈希环上的位置。

来了一个 `key`，首先计算 `hash` 值，并确定此数据在环上的起始位置，从此位置沿环顺时针 "行走"，遇到的第一个节点就是 `key` 所在位置。

如果增减节点，受影响的数据仅仅是此节点到 环空间前/后一个节点 之间的数据，其它不受影响。相对于 `hash` 算法，大大减少了失效 `key` 的数量。



燃鹅，一致性哈希算法在节点太少时，容易因为节点分布不均匀而造成 **数据不均衡** 的问题。为了解决这种热点问题，可以引入虚拟节点机制，将多个虚拟节点映射到物理节点上。查找 `key` 所在分片时，先找到对应的虚拟节点，再对应到实节点。redis cluster，以及 maglev 算法使用的都是这种思路。





### redis cluster 的 hash slot 算法

其实就是虚拟节点的思路，每个虚拟节点叫一个 `slot`。

`redis cluster` 有固定的 `16384` 个 `hash slot`，对每个 `key` 计算 `CRC16` 值，然后对 `16384` 取模，可以获取 `key` 对应的 `hash slot`。

`redis cluster` 中每个 `master` 都会持有部分 `slot`，比如有 3 个 `master`，那么就是每个 `master` 持有 `5000` 多个 `hash slot`。

`hash slot` 让节点的增加和移除变得简单，增加一个节点，就将其他节点的 `slot` 分一部分过去；减少一个节点，就将它的迁移到其他节点上。

`redis cluster` 是服务端分片，增减节点时，通过 `reshard` 命令，可以自动迁移数据。





#### 参考

> [Horizontal scaling in/out techniques for redis cluster](https://iamvishalkhare.medium.com/horizontal-scaling-in-out-techniques-for-redis-cluster-dcd75c696c86)
