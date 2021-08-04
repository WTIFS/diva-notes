## dictEntry
哈希表里的 `key` 和 `value`，是封装为 `dictEntry` 对象存储的。

```c
//24字节
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    struct dictEntry *next;     // 指向下个哈希表节点，形成链表
} dictEntry;
```





## dict 哈希表

`redis` 里的 `hashtable` 使用了链表法解决哈希冲突。

```c
// 哈希表
// 每个字典都使用两个哈希表子结构，以实现渐进式 rehash 。
typedef struct dict {
    dictType *type; // 类型特定函数
    void *privdata; // 私有数据
    dictht ht[2];   // ht=hash table，每个dict有俩，增量扩容时使用
    
    // 搬迁进度
    // 因为是渐进式的哈希，数据的迁移并不是一步完成的，所以需要有一个索引来标记当前的搬迁进度
    // rehash 是 以bucket(桶)为基本单位进行渐进式的数据迁移的，每步完成一个 bucket 的迁移，直至所有数据迁移完毕
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
} dict;

// 哈希表子结构
typedef struct dictht {
    // 哈希表数组
    // 可以看作是：一个哈希表数组，数组的每个项是entry链表的头结点（链地址法解决哈希冲突）
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```

![image-20210804164931040](assets/image-20210804164931040.png)



在 `redis` 中，扩展或收缩哈希表需要将 `ht[0]` 里面的所有键值对 `rehash` 到 `ht[1]` 里面， 但是， 为了避免 `rehash` 对服务器性能造成影响，这个 `rehash` 动作并不是一次性、集中式地完成的， 而是分多次、渐进式地完成的。 这个跟 `Golang` 里 `map` 的扩容几乎是一样的。





## 字段初始化

在 `redis` 中字典中的 `hash` 表采用的是延迟初始化策略。创建字典的时候并没有为哈希表分配内存，只有当第一次插入数据时，才真正分配内存





## 哈希表渐进式 rehash 的详细步骤
1. 为 `ht[1]` 分配两倍原哈希表的空间， 让字典同时持有 `ht[0]` 和 `ht[1]` 两个哈希表。
2. 在字典中维持一个索引计数器变量 `rehashidx` ， 并将它的值设置为 `0` ， 表示 `rehash` 工作正式开始。
3. 在 `rehash` 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 `ht[0]` 哈希表在 `rehashidx` 索引上的所有键值对 `rehash` 到 `ht[1]` ， 当 `rehash` 工作完成之后， 程序将 `rehashidx` 的值加 `1`。期间 `hash[0]` 停写。
4. 随着字典操作的不断执行， 最终在某个时间点上， `ht[0]` 的所有键值对都会被 `rehash` 至 `ht[1]` ， 这时将 `rehashidx` 属性设为 `-1` ， 表示 `rehash` 已完成。





## 触发条件

1. 哈希表中保存的 `key` 数量超过了哈希表的大小
2. 没有在执行 `BGSAVE / BGREWRITEAOF` 命令 且 `keys / size >= 5`
3. 没有在执行 `BGSAVE / BGREWRITEAOF` 命令 且 `keys / size <= 0.1`





## 触发时机

1.  在 `redis` 中每一个增删改查命令中都会判断数据库字典中的哈希表是否正在进行搬迁，如果是则帮助执行一次
2.  如果服务器比较空闲，`redis` 数据库将很长时间内都一直使用两个哈希表。所以也会有定时任务周期性检查，如果发现有字典正在进行搬迁，则会花费1毫秒的时间协助搬迁





## 渐进式 rehash 执行期间的哈希表操作

因为在进行渐进式 `rehash` 的过程中， 字典会同时使用 `ht[0]` 和 `ht[1]` 两个哈希表， 所以在渐进式 `rehash` 进行期间， 字典的删改查等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 `ht[0]` 里面进行查找， 如果没找到的话， 就会继续到 `ht[1]` 里面进行查找。

> 这步和 `Go` 的 `map` 不一样。`Go` 里读正在搬迁的 `map` 时，会先进行搬迁，搬迁后直接读新桶，而不是同时查新老两个桶。

另外， 在渐进式 `rehash` 执行期间， 新添加到字典的键值对一律会被保存到 `ht[1]` 里面， 而 `ht[0]` 则不再进行任何添加操作： 这一措施保证了 `ht[0]` 包含的键值对数量会只减不增， 并随着 `rehash` 操作的执行而最终变成空表。





## 好处

渐进式 `rehash` 的好处在于它采取分而治之的方式， 将 `rehash` 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 `rehash` 而带来的庞大计算量。





## 问题

渐进式 `rehash` 避免了`redis` 阻塞，可以说非常完美，但由于在 `rehash` 时，需要分配一个新的 `hash` 表，在 `rehash` 期间，同时有两个 `hash` 表在使用，会使得 `redis` 内存使用量瞬间突增，在 `redis` 满容状态下由于 `rehash` 会导致大量 `key` 驱逐。