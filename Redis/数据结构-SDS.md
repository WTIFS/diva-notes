# SDS
- `redis` 中对`string`值的存储，并没有使用C中的 `char*`，而是使用了 `SDS (simple dynamic string)`
- `SDS` 结构类似 `Go` 里的切片，有 `3` 个字段，总长度 `len`，空闲长度 `free`，以及实际字符串数组 `char*`
- 相对于 `C` 的 `char*`，有如下优势：
  - 二进制安全的（不用原字符串 + `\0` 的方式表示结束）
  - 能在 `O(1)` 的时间获取字符串长度
  - 预留了额外空间，追加字符串时可能不需要重新分配空间；（惰性释放，内存紧张时也会释放）

```c
// 空间占用为3B + 字符串长度
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;         /* 已使用的长度 used */
    uint8_t alloc;       /* 分配的总长 excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```





### SDS 的编码

- `SDS` 的实现有 `2` 种，在长度特别短时，使用 `embstr` 形式存储，而当长度超过 `44` 字节时，使用 `raw` 形式存储。
- 两者其实都是`header` + `char*`的组合。
- `embstr` 比 `raw` 少一次内存空间分配，它把 `header` 和 字符串保存在同一块连续的内存里了。而 `raw` 要申请额外内存保存字符串。

> 为啥是 `44` 字节？
>
> `64` 字节，减去 `RedisObject` 头信息 `19` 字节，再减去 `3` 字节 `SDS` 头信息，剩下 `45` 字节，再去除 `\0` 结尾。这样最后可以存储 `44` 字节。





### Append 函数的实现
新字符串长度 = (原有长度 + 新长度) + 预分配空间

预分配空间的算法如下：

```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    newlen = (len+addlen);         // 追加后的长度
    if (newlen < SDS_MAX_PREALLOC) // 1MB
        newlen *= 2;               // 上次用了len长度的空间，那么下次程序可能也会用len长度的空间，所以redis就为你预分配。这个有点谜
    else
        newlen += SDS_MAX_PREALLOC;
}
```

