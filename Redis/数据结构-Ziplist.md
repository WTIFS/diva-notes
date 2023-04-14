`ziplist` 是 Redis 中的一种紧凑型、压缩型、双向链表结构。

在 `ziplist` 中，每个节点 `entry`由两部分字段组成：头部信息和数据信息。

- 头部信息包括节点的长度、前一个节点的长度和节点的类型
- 数据信息指向实际的数据。



## 结构

```go
type ziplist struct {
    zlbytes int32     // ziplist的长度（单位: 字节)，是一个32位无符号整数
    zltail  int32     // ziplist最后一个节点的偏移量，反向遍历ziplist或者pop尾部节点的时候有用。
    zlle    int16     // ziplist的节点（entry）个数
    entry   []entry   // 节点
    zlend   int8      // 值为0xFF，用于标记ziplist的结尾
}
```



虽然 redis 定义了节点 [zlentry](https://github.com/redis/redis/blob/6.0/src/ziplist.c) 结构体，但却没有用 `zlentry` 结构来存储节点。因为这个结构存小整数或短字符串太浪费空间，不符合压缩列表提高内存利用率的设计目的，因此，在 redis 中，并没有使用 `zlentry` 结构，而是定义了宏来表示压缩列表的节点。 `entry` 的实际结构如下：

```go
type entry struct {
    prevlength int8         // 上一个节点的长度，为了方便反向遍历 ziplist
    encoding    string      // 当前节点的编码规则
    data        interface{} // 当前节点的值，可以是数字或字符串
}
```

`previous_entry_length`：前一个节点的长度，占 `1Byte` 或 `5Byte`

- 如果前一个节点的长度小于 `254Byte` ，则需要 `1Byte` 来保存前一个节点的长度
- 如果前一个节点的长度 `>= 254Byte` ，则需要 `5Byte` 来保存前一个节点的长度，第一个 `Byte` 固定为 `0xfe(254)`，后四个 `Byte` 表示前一个节点的长度。用 `254` 不用 `255(11111111)` 作为分界是因为`255` 是 `zlend` 的值，它用于判断 `ziplist` 是否到达尾部。



`encoding` 编码的设计较为复杂，这里简单写下。对于 `encoding` 的前两位：

- `11` 表示为整数
- `00` 表示为字符串
- 整数节点 `encoding` 永远是 `8` 位，字符串节点的 `encoding` 可以有 `8` 位、`16` 位、`40` 位三种长度

所以在 `ziplit.c` 里可以看到大量如下的代码：
```c
if (encoding == ZIP_INT_8B) {
    ret = ((int8_t*)p)[0];
} else if (encoding == ZIP_INT_16B) {
    memcpy(&i16,p,sizeof(i16));
    memrev16ifbe(&i16);
    ret = i16;
}
```

![74be0cf64110c18388dbc04c83efc4f5.png](assets/zipentry.jpeg)





`ziplist` 用在 `Zset` 和 `Quicklist` 中。使用时的结构是这样的：

![aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmpp](assets/aHR0cHM6Ly91cGxvYWQtaW1hZ2VzLmpp.jpg)





## 插入/删除元素

- 向中部插入/删除某个节点时，需要更新 `next` 节点的 `pre_entry_length`
- `next` 节点的 `pre_entry_length` 变长或变短可能又导致节点本身长度改变，进而导致 `next.next.pre_entry_length` 需要变更
- 这就是说， 在某个/某些节点的前面添加新节点之后， 程序必须沿着路径挨个检查后续的节点，是否满足新长度的编码要求
- 不过，因为只有在新添加节点的后面有连续多个长度接近 `254` 的节点时，这种连锁更新才会发生， 所以可以普遍地认为，这种连锁更新发生的概率非常小