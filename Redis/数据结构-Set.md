# Set 集合

`REDIS_SET`（集合）是 `SADD`、`SRANDMEMBER` 等命令的操作对象，可以对元素进行去重，其内部使用了 `Hashtable` 和 `Intset` 两种数据结构，即  `REDIS_ENCODING_INTSET` 和 `REDIS_ENCODING_HT` 两种方式编码。



## 编码的选择

第一个添加到集合的元素， 决定了创建集合时所使用的编码：
- 如果第一个元素可以表示为 `long long` 类型值（就是整数）， 那么集合的初始编码为 `REDIS_ENCODING_INTSET` 。
  - 改编码所用结构实际是有序数组。更节省空间。
  - 查找时使用的是二分查找，时间复杂度为`O(lgn)`，不是`O(1)`
- 否则，集合的初始编码为 `REDIS_ENCODING_HT` 。



## 编码的切换

如果一个集合使用 `REDIS_ENCODING_INTSET` 编码， 那么当以下任何一个条件被满足时， 这个集合会被转换成 `REDIS_ENCODING_HT` 编码：
- `intset` 保存的整数值个数超过 `server.set_max_intset_entries` （默认值为 512 ）。
- 试图往集合里添加一个非整数元素。



## IntSet 结构

`intset` 实际上就是对有序数组封装了下：

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t   contents[];
} intset;
```





## 和 Hashtable 的关系

使用 `REDIS_ENCODING_HT` 编码时，`Set `实际上就是一个 `value` 为`NULL`的`Hashtable`。使用的数据结构就是 `dict`。

放入 `key` 后，保存  `value` 时所用的方法正是 `Hashtable` 的 `dictSetVal` 方法：`dictSetVal(ht,de,NULL)`。只不过 `Set` 传的第三个参数值为 `NULL`。

