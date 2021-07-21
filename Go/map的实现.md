# map hash 冲突的常见解决方案

1. 开放地址法
   - 放入元素，如果发生冲突，就往后找没有元素的位置
2. 链表法
   - `hashcode` 相同的串成链表。`JAVA` 采用的方法
3. 再哈希
   - 用另一个方法计算 `hashcode`。布谷鸟过滤器使用的方法
4. 公共溢出区
   - 发生冲突的元素，一律填入溢出表

`Go` 使用的是链表法 + 公共溢出区方法





# map 数据结构

```go
// src/runtime/hashmap.go

type hmap struct {
    count     int    // 元素个数，调用 len(map) 时，直接返回此值
    flags     uint8  // 并发读写的状态标志，如果 =1 时读写这个 map，会 panic
    B         uint8  // log2(桶数)，也就是说 buckets 数组的长度就是 2^B
    noverflow uint16 // 溢出的个数
    hash0     uint32 // 哈希种子

    buckets    unsafe.Pointer // 指向 buckets 数组，大小为 2^B
    oldbuckets unsafe.Pointer // 旧桶的地址，用于扩容
    nevacuate  uintptr        // 指示扩容进度，小于此地址的 buckets 迁移完成
    overflow *[2]*[]*bmap     // 公共溢出区。有两个，分别是 buckets 和 oldbuckets 的公共溢出区
}
```

`buckets` 是一个指针，最终它指向的是一个结构体：

```golang
type bmap struct {
	tophash [bucketCnt]uint8
}
```

但这只是表面 (`src/runtime/hashmap.go`) 的结构，编译期间会给它加料，动态地创建一个新的结构：

```golang
type bmap struct {
    topbits  [8]uint8     // hash值的高8位，用于排序
    keys     [8]keytype   // 8个key
    values   [8]valuetype // 8个value
    pad      uintptr
    overflow uintptr      // 如果桶里的元素超过8个，那就需要再构建一个桶，这个字段会指向这个新桶，相当于多个桶用链表串起来
}
```

`bmap` 就是我们常说的桶，桶里面会最多装 8 个 `key`。在桶内，又会根据 `key` 计算出来的 `hash` 值的高 8 位来查找 `key`。

如果桶里的元素要超过 8 个了，这时候需要在桶后面挂上 `overflow bucket`。当然，也有可能是在 `overflow bucket` 后面再挂上一个 `overflow bucket`。这就说明，太多 `key` hash 到了此 `bucket`。

注意到 `key` 和 `value` 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。这样的好处是在某些情况下可以避免内存对齐，节省空间。

比如 `map[int64]int8` 这样的 `map`，如果按照 `key/value/key/value/...` 这样的模式存储，那在每一个 `key/value` 对之后都要额外 `padding` 7 个字节；而 `key/key/.../value/value/...`这种形式则只需要在最后添加 `padding`。





![img](assets/v2-0178a76f87bb68fd7a645e6885e17525_b.jpg)





# 查找

`key` 经过哈希计算后得到哈希值，共 64 个 `bit` 位（32位机不讨论了），计算它到底要落在哪个桶时，只会用到最后 `B` 个 `bit` 位。还记得前面提到过的 `B` 吗？如果 `B = 5`，那么桶的数量，也就是 `buckets` 数组的长度是 `2^5 = 32`。

例如，现在有一个 `key `经过哈希函数计算后，得到的哈希结果是：

```shell
hash高5位   hash                                                  hash低5位
10010111  | 000011110110110010001111001010100010010110010101010 │ 01010
```

用最后的 5 个 `bit` 位 `01010`，也就是 `10` 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

再用哈希值的高 `8` 位，快速比较，找到此 `key` 在 `bucket` 中的位置。如果找到一样的，再进一步比较 `key` 的原值。

如果在 `bucket` 中没找到，并且 `overflow` 不为空，还要继续去 `overflow bucket` 中寻找。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 计算出高 8 位的 hash
	// 相当于右移 56 位，只取高8位
	top := uint8(hash >> (sys.PtrSize*8 - 8))
    
    // 扩容用的，top < minTopHash时，表示迁移相关，加上 minTopHash，以区分正常的 top hash 值和表示状态的值。
	if top < minTopHash {
		top += minTopHash
	}
    
    for {
            // 遍历 8 个 bucket
            for i := uintptr(0); i < bucketCnt; i++ {

                // tophash 不匹配，继续
                if b.tophash[i] != top {
                    continue
                }
                // tophash 匹配，定位到 key 的位置
                k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
               
                // 进一步比较 key 原值
                if alg.equal(key, k) {
                    // 定位到 value 的位置
                    v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
                    return v
                }
            }

            // bucket 找完（还没找到），继续到 overflow bucket 里找
            b = b.overflow(t)
            // overflow bucket 也找完了，说明没有目标 key
            // 返回零值
            if b == nil {
                return unsafe.Pointer(&zeroVal[0])
            }
        }
    }
}
```





#### 参考

> [Stefno - 深度解密Go语言之 map](https://zhuanlan.zhihu.com/p/66676224)

