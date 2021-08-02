# map hash 冲突的常见解决方案

1. 开放地址法
   - 放入元素，如果发生冲突，就往后找没有元素的位置
2. 链表法
   - `hashcode` 相同的串成链表。`Go、JAVA` 采用的方法
3. 再哈希
   - 用另一个方法计算 `hashcode`。布谷鸟过滤器使用的方法
4. 公共溢出区
   - 发生冲突的元素，一律填入溢出表





# map 数据结构

```go
// src/runtime/hashmap.go

type hmap struct {
    count     int    // 元素个数，调用 len(map) 时，直接返回此值
    flags     uint8  // 并发读写的状态标志，如果 =1 时读写这个 map，会 panic
    B         uint8  // log2(桶数)，也就是说 buckets 数组的长度就是 2^B
    noverflow uint16 // 溢出的 bucket 个数
    hash0     uint32 // 哈希种子

    buckets    unsafe.Pointer // 指向 buckets 数组，大小为 2^B
    oldbuckets unsafe.Pointer // 旧桶的地址，用于扩容
    nevacuate  uintptr        // 指示扩容进度，小于此地址的 buckets 迁移完成
    overflow *[2]*[]*bmap     // map 里不含指针时，用这个存 buckets 和 oldbuckets 的溢出区，保证溢出区在扫描时能被扫到
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

`key` 经过哈希计算后得到哈希值，共 `64` 个 `bit` 位（`32` 位机不讨论了），计算它到底要落在哪个桶时，只会用到最后 `B` 个 `bit` 位。还记得前面提到过的 `B` 吗？如果 `B = 5`，那么桶的数量，也就是 `buckets` 数组的长度是 `2^5 = 32`。

例如，现在有一个 `key `经过哈希函数计算后，得到的哈希结果是：

```shell
hash高5位   hash                                                  hash低5位
10010111  | 000011110110110010001111001010100010010110010101010 │ 01010
```

用最后的 `5` 个 `bit` 位 `01010` 作为桶下标，也就是 `10` 号桶。

再用哈希值的高 `8` 位，快速比较，找到此 `key` 在 `bucket` 中的位置。如果找到一样的，再进一步比较 `key` 的原值。

如果在 `bucket` 中没找到，并且 `overflow` 不为空，还要继续去 `overflow bucket` 中寻找。

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// 计算出高 8 位的 hash
	// 相当于右移 56 位，只取高8位
	top := uint8(hash >> (sys.PtrSize*8 - 8))
    
    // 扩容用的，top < minTopHash时，表示迁移相关，+= minTopHash，以区分正常的 top hash 值和表示状态的值。
	if top < minTopHash {
		top += minTopHash
	}
    
    // 遍历所有 bucket
    for {
            // 遍历 bucket 的 8 个位置
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



`Go` 中的 `map` 有两种语法：带 `comma` 和 不带 `comma`。这其实是编译器在背后做的工作：分析代码后，将两种语法对应到底层两个不同的函数。

```go
// src/runtime/hashmap.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```



另外，根据 `key` 的不同类型，编译器还会将查找、插入、删除的函数用更具体的函数替换，以优化效率。由于提前知晓了 `key` 的类型，所以内存布局是很清楚的，能节省很多操作，提高效率。

```go
// src/runtime/hashmap_fast.go
key的类型  函数
uint32	   mapaccess1_fast32(t *maptype, h *hmap, key uint32) unsafe.Pointer
uint64	   mapaccess1_fast64(t *maptype, h *hmap, key uint64) unsafe.Pointer
string	   mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer
```



如果遇到 `map` 正在扩容的情况，那查找之前还要先判读一下 `key` 是在新桶里还是旧桶里。

```go
m := 2^B -1 // mask
b := (*bmap)(add(h.buckets, (hash&m) + uintptr(t.bucketsize))
```





# 扩容

### 时机

可以看到，同一个桶内，所有元素是组成链表连接起来的。如果一个某个桶内的链表元素过多，会导致查询性能下降。

因此，需要有一个指标来衡量前面描述的情况，这就是**负载因子**。`Go` 里的 **负载因子** = **元素个数 / 桶数**

```go
loadFactor := count / (2^B)
```

扩容的时机：在向 `map` 插入新 `key` 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：

1. 装载因子超过阈值，源码里定义的阈值是 `6.5`。
2. 溢出的桶数量过多（作为对第一点的补充，插入很多元素、再删除时，可能导致元素本身数量不多，但 `overflow` 桶很多）
   1. `B < 15 && noverflow >= 2^B`
   2. `B >= 15 && noverflow >= 2^15`

这两种条件下都会发生扩容。但是扩容的策略并不相同，毕竟两种条件应对的场景不同。

1. 对于条件 1，元素太多，而 `bucket` 数量太少，很简单：将 `B` 加 `1`，新申请一个 `2^(B+1)` 的 `bucket`，`bucket` 数量直接变成原来的 `2` 倍。于是，就有新老 `bucket` 了。注意，这时候元素都在老 `bucket` 里，还没迁移到新的 `bucket` 来。这种扩容叫 **增量扩容**。

2. 对于条件 2，其实元素没那么多，但是 `overflow bucket` 数特别多，说明很多 `bucket` 都没装满。解决办法就是开辟一个新 `bucket` 空间，将老 `bucket` 中的元素移动到新 `bucket`，使得同一个 `bucket` 中的 `key` 排列地更紧密。这种叫 **等量扩容**。严格上说其实不算扩容，算整理碎片。

由于扩容需要将原有的 `key/value` 重新搬迁到新的内存地址，如果有大量的 `key/value` 需要搬迁，会非常影响性能。因此 `Go map` 的扩容采取了 **渐进式扩容** 的方式，类似 `redis` 的扩容，原有的 `key` 并不会一次性搬迁完毕。

> 为啥负载因子是 6.5 ?
>
> 太小会导致极易触发扩容，造成空间浪费；太多会导致极不易触发扩容，造成 `overflow` 过多。
>
> 事实上作者测试了各负载因子下的 平均 `overflow` 数、命中率、`miss` 率等指标，最终取了一个中间数 6.5。



### 扩容过程

扩容其实分为两步：`hashGrow()` 扩容 和 `growWork()` 搬迁。

 `hashGrow()` 函数实际上并没有真正地搬迁，它只是分配好了新的 `buckets`，并将老的 `buckets` 挂到了 `oldbuckets` 字段上。

```go
func hashGrow(t *maptype, h *hmap) {
    bigger := uint8(1)
    if !overLoadFactor(int64(h.count), h.B) {  // 判断是增量扩容还是等量扩容
        bigger = 0
        h.flags |= sameSizeGrow
    }
        
    oldbuckets := h.buckets                           // 将buckets赋值给oldbuckets
    newbuckets := newarray(t.bucket, 1<<(h.B+bigger)) // 申请一个新的、容量为两倍的 bucket 数组
    flags := h.flags &^ (iterator | oldIterator)
    if h.flags&iterator != 0 {
        flags |= oldIterator
    }
   
    // 更新hmap的变量
    h.B += bigger             // 更新 B           
    h.flags = flags
    h.oldbuckets = oldbuckets // 将buckets赋值给oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0
    // 设置溢出桶
    if h.overflow != nil {           // overflow[1] 表示旧桶，overflow[0] 表示新桶
        if h.overflow[1] != nil {    // 如果已经有旧桶，说明有别的协程在扩容，抛出
            throw("overflow is not nil")
        }
		
        h.overflow[1] = h.overflow[0] // 赋值给旧桶
        h.overflow[0] = nil
    }
}
```





### 搬迁过程

搬迁的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在 `mapassign` 和 `mapdelete` 函数中。

也就是插入或修改、删除 `key` 的时候，都会尝试进行搬迁 `buckets` 的工作。先检查 `key` 所在的 `oldbuckets` 是否搬迁完毕，如果没有，则先对其进行搬迁。然后再检查其他搬迁状态过程中的桶，如果有，协助进行搬迁。

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    evacuate(t, h, bucket&h.oldbucketmask()) // 搬迁旧桶，这样assign和delete都直接在新桶集合中进行
    if h.growing() {
        evacuate(t, h, h.nevacuate)          // 再协助搬迁一次其他桶
    }
}
```



接下来，我们集中所有的精力在搬迁的关键函数 `evacuate`：

```golang
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))) // 定位老的 bucket 地址
	newbit := h.noldbuckets()                                        // 结果是 2^B，如 B = 5，结果为32
	alg := t.key.alg                                                 // key 的哈希函数
	
	if !evacuated(b) {            // 如果 b 没有被搬迁过
		var (
			x, y   *bmap          // 表示 bucket 移动的目标地址
			xi, yi int            // 指向 x, y 中的 key/val
			xk, yk unsafe.Pointer // 指向 x, y 中的 key
			xv, yv unsafe.Pointer // 指向 x, y 中的 value
		)
        
		// 默认是等 size 扩容，前后 bucket 序号不变，使用 x 来进行搬迁
		x = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		xi = 0
		xk = add(unsafe.Pointer(x), dataOffset)
		xv = add(xk, bucketCnt*uintptr(t.keysize))

		// 如果不是等 size 扩容，前后 bucket 序号有变，使用 y 来进行搬迁
		if !h.sameSizeGrow() {
			y = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize))) // y 代表的 bucket 序号增加了 2^B
			yi = 0
			yk = add(unsafe.Pointer(y), dataOffset)
			yv = add(yk, bucketCnt*uintptr(t.keysize))
		}

		// 迁移老的 b 及其 overflow 链表
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))

			// 遍历 bucket 中的所有 cell
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
				if top == empty { // 如果 cell 为空，即没有 key，直接标志它被搬迁过
					b.tophash[i] = evacuatedEmpty
					continue
				}

				k2 := k // 如果 key 是指针，则解引用
				if t.indirectkey {
					k2 = *((*unsafe.Pointer)(k2))
				}

				useX := true                                       // 默认使用 X，等量扩容
				if !h.sameSizeGrow() {                             // 如果不是等量扩容
					hash := alg.hash(k2, uintptr(h.hash0))         // rehash
					
                    // rehash的值，由于 B 是 *=2，因此只有 newbit 这位上可能和原 hash 不一样，一样就留在原桶；否则移入新桶
                    useX = hash&newbit == 0                        
				}

				// 如果 key 搬到 X 部分，即等量扩容的逻辑
				if useX {
					b.tophash[i] = evacuatedX       // 标志老的 cell 的 top hash 值，表示搬移到 X 部分
					if xi == bucketCnt {            // 如果 xi 等于 8，说明要溢出了
						newx := h.newoverflow(t, x) // 新建一个 bucket
						x = newx
						xi = 0                                     // xi 从 0 开始计数
						xk = add(unsafe.Pointer(x), dataOffset)    // xk 表示 key 要移动到的位置
						xv = add(xk, bucketCnt*uintptr(t.keysize)) // xv 表示 value 要移动到的位置
					}
					
					x.tophash[xi] = top             // 设置 top hash 值
					if t.indirectkey {              // key 是指针
						*(*unsafe.Pointer)(xk) = k2 // 将原 key（是指针）复制到新位置
					} else {
						typedmemmove(t.key, xk, k)  // 将原 key（是值）复制到新位置
					}
                    
					if t.indirectvalue {
						// value 是指针，操作同 key
						// ...
					}
					
					xi++ // 定位到下一个 cell
					xk = add(xk, uintptr(t.keysize))
					xv = add(xv, uintptr(t.valuesize))
				} else { 
                    // key 搬到 Y 部分，即增量扩容的逻辑，操作同 X 部分
					// ...
				}
			}
		}
        
		// 如果没有协程在使用老的 buckets，就把老 buckets 清除掉，解除引用，清除内存，以便可以GC到
		if h.flags&oldIterator == 0 {
			memclrHasPointers(add(unsafe.Pointer(b), dataOffset), uintptr(t.bucketsize)-dataOffset)
		}
	}

	// 更新搬迁进度
	if oldbucket == h.nevacuate {   // 如果此次搬迁的 bucket 等于当前进度
		h.nevacuate = oldbucket + 1 // 进度加 1
		// Experiments suggest that 1024 is overkill by at least an order of magnitude.
		// Put it in there as a safeguard anyway, to ensure O(1) behavior.
		// 尝试往后看 1024 个 bucket
		stop := h.nevacuate + 1024
		if stop > newbit {
			stop = newbit
		}
		// 寻找没有搬迁的 bucket
		for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
			h.nevacuate++
		}
		
		// 现在 h.nevacuate 之前的 bucket 都被搬迁完毕
		
		// 如果所有的 buckets 搬迁完毕，将老的清除
		if h.nevacuate == newbit {       
			h.oldbuckets = nil 			  // 清除老的 buckets
			if h.extra != nil {
				h.extra.overflow[1] = nil // 清除老的 overflow bucket
			}
			h.flags &^= sameSizeGrow      // 清除正在扩容的标志位
		}
	}
}
```



前面说过，扩容分两种情况：有新建 `bucket`，和没有新建 `bucket`。

对于条件 `2`，由于 `bucket` 数量不变， 不需要 `rehash`。

对条件 `1` 则需要重新计算 `hash`。新的 `buckets` 数量是之前的一倍，因此只需要看新的 `B` 的最高位。例如，原来 `B = 5`，计算出 `key` 的哈希后，只用看它的低 `5` 位。扩容后，`B` 变成了 `6`，因此需要多看一位，它的低 `6` 位决定 `key` 落在哪个 `bucket`。

因此，某个 `key` 在搬迁前后 `bucket` 序号可能和原来相等，也可能是相比原来加上 `2^B`（原来的 `B` 值），取决于 `hash` 值第 `B` 位是 `0` 还是 `1`。





## 遍历

理解了上面 `bucket` 序号的变化，我们就可以回答另一个问题了：为什么遍历 `map` 是无序的？

1. `map` 根本就没有维护 `key` 的顺序，计算 `key` 的桶时是用 `hash` 算的，本来有序的 `key`，`hash` 后就无序了。
2. `map` 在扩容后，会发生 `key` 的搬迁，原来落在同一个 `bucket` 中的 `key`，搬迁后，有些 `key` 就要远走高飞了（`bucket` 序号加上了 `2^B`）。因此，遍历、修改`map`、再遍历，两次得到的遍历顺序就可能不一样了。

当然，`Go` 做得更绝，遍历 `map` 时，并不是固定地从 `0` 号 `bucket` 开始遍历，而是通过 `fastrand` 算了个随机数，从一个随机 `bucket` 开始，并且是从这个 `bucket` 的随机 `cell` 开始遍历。特意设计成了无序迭代的结果。





# 写

对 `key` 计算 `hash` 值，根据 `hash` 值按照之前的流程，找到要赋值的位置。源码大体和之前的类似，核心还是一个双层循环，外层遍历 `bucket` 和它的 `overflow bucket`，内层遍历整个 `bucket` 的各个 `cell`。

函数首先会检查 `map` 的标志位 `flags`。如果 `flags` 的写标志位此时被置 `1` 了，说明有其他协程在执行写操作，进而导致程序 `panic`。这也说明了 `map` 对协程是不安全的。

通过前文我们知道扩容是渐进式的，如果 `map` 处在扩容的过程中，那么这次会先协助扩容，完了才向新 `bucket` 里写数据。





# 删

流程和写是一样的，只不过对指针就赋值 `nil`，对非指针就调用 `typedmemclr`。

删除仅仅是将对应的 `slot` 设置为 `empty`，并没有减少内存。





# 总结

总结一下，`Go` 语言中，通过哈希查找表实现 `map`，用链表法解决哈希冲突。利用将 8个 `key` / 8个`value` 依次防止的做饭减少了对齐所需的空间。

通过 `key` 的哈希值将 `key` 散落到不同的桶中。比如说有 `2^5=32` 个桶，就用哈希值的低 `5` 位判断落入哪个桶。

当向桶中添加了很多 `key`，造成元素过多（每个桶内的元素数超过 `6.5`），或者溢出桶太多（溢出桶数量超过桶数量），就会触发扩容。

- 对前一种情况，加桶的个数就可以了，对应的是 `2` 倍容量的 **增量扩容**，扩容期间需要 `rehash` 重新分桶。
- 后者是由于大量写入和删除元素造成的数据空洞，只需要重新整理下溢出桶里的数据就可以，称为 **等量扩容**。

扩容过程是渐进的，主要是防止一次扩容需要搬迁的 `key` 数量过多，引发性能问题。触发扩容的时机是增加了新元素，搬迁的时机则发生在赋值/删除期间，每次最多搬迁两个 `bucket`（这里存疑，代码里只看到一个）。





#### 参考

> [Stefno - 深度解密Go语言之 map](https://zhuanlan.zhihu.com/p/66676224)
>
> [健 の 随笔 - 你不知道的Golang map](https://www.cnblogs.com/sunsky303/p/11815172.html)

