# 简介
有序集合 `Zset` 常用来做优先队列。在集合的去重保存元素的基础上，为每个元素加了个**分值**，元素可以按分值排序。

根据数据量，有 `ziplist` 和 `skiplist + hashtable` 两种编码方式。

- 数据量少时，使用 `ziplist`
    - 通过配置文件读取 最大元素数量、最大长度；超过数量或长度，就会换为跳表
    - 查找时是遍历查找的
- 数据量多时，使用跳表  + 哈希表
    - `member` 为 `set` 元素，`value` 为分值
    - 哈希表可以查找 `member` 对应的分值，或判断 `member` 是否存在
    - 跳表可以按范围查找分值内的 `member`



#### 哈希表的作用

通过使用字典结构， 并将 `member` 作为键， `score` 作为值， 有序集可以在 `O(1) ` 复杂度内：
- 检查给定 `member` 是否存在于有序集（被很多底层函数使用）
- 取出 `member` 对应的 `score` 值（实现 `ZSCORE` 命令）



#### 跳表的作用

另一方面， 通过使用跳跃表， 可以让有序集支持以下两种操作 (`O(lgn)`)：
- 根据 `score` 对 `member` 进行定位（被很多底层函数使用）
- 范围性查找和处理操作，高效地实现 `ZRANGE` 、 `ZRANK` 和 `ZINTERSTORE` 等命令的关键



通过同时使用字典和跳跃表， 有序集可以高效地实现 **按成员查找** 和  **按顺序查找** 两种操作。





# 实现

### ziplist 编码

当使用 `ziplist` 作为底层存储结构时候，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员，第二个元素保存元素的分值。

并且压缩列表内的集合元素按分值从小到大的顺序进行排列。

![aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmpp](assets/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmpp.jpg)





### skiplist 编码

```c
// redis.h/zset
typedef struct zset { 
    dict *dict;     // 字典
    zskiplist *zsl; // 跳跃表
} zset;
```

字典的键保存元素的值，字典的值则保存元素的分值；跳跃表节点的 `object` 属性保存元素的成员，跳跃表节点的 `score` 属性保存元素的分值。

这两种数据结构会通过指针来共享相同元素的成员和分值，所以不会产生重复成员和分值，造成内存的浪费。







# 问题

##### Zset只能按一维的Score排序，如何实现多维排序？
将多个维度的列通过一定的方式转换成一个特殊的列，即 `score = function(维度1的rank, 维度2的rank, 维度3的rank)`

相当于维度1相同的情况下，再比较维度2，最后比较维度3



#### 参考

> [隨意的風 - redis缓存数据库中zset数据结构底层算法实现原理：ziplist 和 skiplist](https://blog.csdn.net/Windgs_YF/article/details/103031177)

