### 基本概念

跳表是可以实现二分查找的有序链表，实现原理是将多个链表在纵向串起来，在原有的有序链表上面增加了多级索引，通过索引来实现快速查找。

主要用于有序集合。



### 设计

1. 最低层包含所有的元素
2. 每个索引节点包含两个指针，一个向下，一个向右
3. 插入元素时，先插入最底层，然后以 `1/4` 的概率插入上层（最大层高 = `32`）。这样随机生成它的层高
4. 跳表查询、插入、删除的时间复杂度为 `O(log n)`，与平衡二叉树接近

![img](assets/20160131083150090.jpeg)





### 跳表 VS 二叉树

1. 范围查找更快。二叉树需要不断的中序遍历，性能较差。
2. 跳表更紧凑，省空间。每个节点平均 `1.33` 个指针。二叉树是 `2` 个指针。
  
4. 跳表实现更简单。二叉树增删元素时可能需要调整多层节点；跳表只需调整前后的邻节点。

```mathematica
每个节点的平均层数 = 1*p + 2*p(1-p) + ... + 32*p^31(1-p) = (1-p) * sum[np^(n-1)] = (1-p) * 1/(1-p)^2 = 1/(1-p) = 4/3
```





## 实现

#### 跳表节点 zskiplistNode

```c
// src/t_zset.c
typedef struct zskiplistNode {  
    robj *obj;     // 元素 
    double score;  // 分值用于排序 
    struct zskiplistNode *backward;     // 前驱节点
    struct zskiplistLevel {  
        struct zskiplistNode *forward;  // 后继节点
        unsigned int span;              // 层跨度，记录本节点到下个节点跨过了几个元素
    } level[];                          // 分层数组
} zskiplistNode;
```



#### 插入

插入前由 `hashmap` 来防重

```c
// src/t_zset.c

// zadd key field score，参数里的 ele 是 field
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) { 
	for (i = zsl->level-1; i >= 0; i--) { // 从最上层开始，遍历每层
         while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0))) // scrore相同的，比较字符串值
        {
            x = x->level[i].forward;
        }
        update[i] = x; // 使用 update 数组记录每层 x 插入位置的前驱节点
    }
    
    level = zslRandomLevel(); // 随机生成本次插入的层数，每多一层，概率*0.25
    
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward; // 将 x.next 指向原下一个节点
        update[i]->level[i].forward = x;  
    }
}

// 随机生成层数
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^64 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF)) // 等价于 while (random() < 0.25)
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```



#### 问题

[随机层数的概率为什么是 p=0.25?](https://github.com/redis/redis/pull/3889)

这个值是对空间和时间的权衡。p值约大，查询速度就越快（因为上层索引多），但占用的空间也越多。



#### 参考

> [lz710117239 - redis(五)跳跃表](https://blog.csdn.net/lz710117239/article/details/78408919)
>
> [Why use skip list instead of binary tree?](https://news.ycombinator.com/item?id=1171423)

