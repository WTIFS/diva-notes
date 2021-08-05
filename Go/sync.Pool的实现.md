`sync.Pool` 是 `Golang` 内置的对象池技术，可用于缓存临时对象，避免因频繁建立临时对象所带来的消耗以及对 `GC` 造成的压力。

在许多知名的开源库中，都可以看到 `sync.Pool` 的大量使用。例如，`HTTP` 框架 `Gin` 用 `sync.Pool` 来复用每个请求都会创建的 `gin.Context` 对象。 在 `grpc-Go`、`kubernates` 等也都可以看到 `sync.Pool`的身影。

但需要注意的是，`sync.Pool` 缓存的对象随时可能被无通知的清除，即 `Put` 一个对象进池子后，执行 `Get` 操作可能获取到的是一个新对象，原来那个 `Put` 进去的对象没了。因此不能将 `sync.Pool` 用于存储持久对象的场景。

`sync.Pool` 作为 `Go` 内置的官方库，其设计非常精妙。`sync.Pool` 不仅是并发安全的，而且实现了 lock free，里面有许多值得学习的知识点。

本文将基于 [go-1.16 的源码](https://github.com/golang/go/blob/release-branch.go1.16/src/sync/pool.go) 对 `sync.Pool` 的底层实现一探究竟。





# 基本用法

在正式讲 `sync.Pool` 底层之前，我们先看下 `sync.Pool` 的基本用法。其示例代码如下：

```go
type Test struct {
	A int
}

func main() {
	pool := sync.Pool{
		New: func() interface{} {
			return &Test{
				A: 1,
			}
		},
	}

	testObject := pool.Get().(*Test) // 从池子里获取或新建一个对象
	println(testObject.A)            // print 1 
	pool.Put(testObject)             // 把对象放回池子
}
```

`sync.Pool` 在初始化的时候，需要用户提供一个对象的构造函数 `New`。用户使用 `Get` 来从对象池中获取对象，使用 `Put` 将对象归还给对象池。整个用法还是比较简单的。





# sync.Pool 的底层实现

在讲 `sync.Pool` 之前，我们先聊下 `Golang` 的 `GMP` 调度。在 `GMP` 调度模型中，`M` 代表了系统线程，而同一时间一个 `M` 上只能同时运行一个 `P`。那么也就意味着，从线程维度来看，在 `P` 上的逻辑都是单线程执行的。

`sync.Pool` 就是充分利用了 `GMP` 这一特点。对每个 `sync.Pool{}` 实例，它里面设了一个长度为 `P` 数量的数组 `local`，数组里每个元素对应一个 `P`，保存的是 `P` 独享的本地对象池，这样每个 `P` 处理自己的池子时就不需要加锁了。

```go
type Pool struct {
	noCopy noCopy             // 告诉编译器该对象不可复制

	local     unsafe.Pointer  // 池子数组，长度为 P 的个数，保存的是 P 独享的本地对象池。其元素类型是 poolLocal
	localSize uintptr         // local的大小

	victim     unsafe.Pointer // GC 时上一轮清理前的对象池
	victimSize uintptr        // size of victims array

	New func() interface{}    // 用户提供的创建对象的函数
}
```



再来看下本地对象池 `poolLocal` 的定义，核心结构是一个二维链表：

```go
// 每个 P 都会有一个 poolLocal 的本地
type poolLocal struct {
	poolLocalInternal
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

type poolLocalInternal struct {
	private interface{} // 单实例变量，优先存取这个变量，如果该变量可以满足情况，则不再深入进行其他的复杂操作。
	shared  poolChain   // 对象池子链表，是个二维链表
}

type poolChain struct { // 链表结构
	head *poolChainElt  // 头节点
	tail *poolChainElt  // 尾节点
}

type poolChainElt struct {   // 节点结构
	poolDequeue              // 数组
	next, prev *poolChainElt // 节点前后指针
}

type poolDequeue struct {    // 数组结构
	headTail uint64          // 头尾指针
	vals []eface             // 数组
}
```



<img src="https://www.cyhone.com/img/sync-pool/local_pool.png" alt="img" style="zoom: 80%;" />

为什么 `poolChain` 是这么一个链表 + 数组 的复杂结构呢？简单的用一个链表不行吗？

使用数组是因为它有以下优点：

1. 预先分配好内存，且分配的内存项可不断复用。
2. 数组是连续内存结构，非常利于 `CPU Cache`。在访问 `poolDequeue` 某一项时，其附近的数据项都有可能加载到统一 `Cache Line`中，访问速度更快。



我们再注意看一个细节，在 `poolDequeue` 的定义中，`head` 和 `tail` 并不是独立的两个变量，只有一个 `uint64` 的 `headTail` 变量。

这是因为 `headTail` 变量将 `head` 和 `tail` 打包在了一起：高 `32` 位是 `head` 变量，低 `32` 位是 `tail` 变量。

这样处理并发时时，只需要 `CAS` 修改这一个变量，不需要锁两个变量。





## Put 的实现

我们看下 `Put` 函数的实现。通过 `Put` 函数我们可以把不用的对象放回或者提前放到 `sync.Pool` 中。`Put` 函数的代码逻辑如下：

```go
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}

	l, _ := p.pin()

	if l.private == nil { // 优先使用 private 变量
		l.private = x
		x = nil
	}

	if x != nil {         // 如果 private 已被使用，则使用 poolLocal 池子
		l.shared.pushHead(x)
	}

	runtime_procUnpin()
}
```

1. 在 `Put` 函数中首先调用了 `pin()`。`pin` 函数非常重要，它有三个作用：
    1. 初始化或者重新创建 `local` 数组。 当 `local` 数组为空，或者和当前的 `P` 数量不一致时，将触发重新创建 `local` 数组。

    2. 取当前 `P` 对应的本地缓存池 `poolLocal`。代码逻辑很简单，就是从 `local` 数组中根据下标取元素。

    3. 防止当前 `P` 被抢占。这点非常重要。在 `Go 1.14` 以后，`Golang` 实现了抢占式调度：一个协程占用 `P` 时间过长，将会被调度器强制挂起。如果一个协程在 `Put` 或者 `Get` 期间被挂起，有可能下次恢复时，绑定就不是上次的 `P` 了，那读写的就是别的 `P` 的池子了。因此，这里使用了 `runtime` 包里面的 `procPin`，暂时不允许 `P` 被抢占。

2. 接着，`Put` 函数会优先设置当前 `poolLocal` 的私有变量 `private`。

3. 如果私有变量之前已经设置过了，那就只能往当前 `P` 的本地缓存池 `poolChain` 里面写了。这步骤可以结合上面 `poolLocal` 的图看
	1. 如果 `HEAD` 不存在 ，则新建一个 `buffer` 长度为 `8` 的 `poolDequeue`，并将对象放置在里面。
	2. 如果 `HEAD` 存在，且 `buffer` 尚未满，则将元素直接放置在 `poolDequeue` 中。
	3. 如果 `HEAD` 存在，但 `buffer` 满了，则新建一个新的 `poolDequeue`，长度为上个 `HEAD` 的 `2` 倍。并将 `HEAD` 指向新的元素。

`Put` 的过程比较简单，整个过程不需要和其他 `P` 的 `poolLocal` 进行交互。





## Get 的实现

`Get` 的实现更为复杂。不仅涉及到对当前 `P` 本地对象池的操作，还涉及对其他 `P` 的本地对象池的对象窃取。

```go
func (p *Pool) Get() interface{} {
	l, pid := p.pin()

	x := l.private                // 优先读取 private 变量
	l.private = nil

	if x == nil {
		x, _ = l.shared.popHead() // 然后读取本地对象池

		if x == nil {
			x = p.getSlow(pid)    // 最后尝试从别的 P 偷 share 对象池
		}
	}
	runtime_procUnpin()

	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

1. 和 `Put` 一样，这里会优先取 `private` 变量
2. 再从 `poolLocal` 对象里取。
	- 这里遍历 `poolLocal` 时，是从头向后遍历

3. 不同的是，如果 `poolLocal` 里取不到，会通过 `getSlow` 函数，从别的 `P` 的缓存池里偷窃对象。
	- 这里遍历别的 `P` 的 `poolDequeue` 时，是从后往前遍历，这样可以尽量减少并发冲突。
	- 这里还会顺便做下 `GC`，如果遍历别的 `P` 的 `poolDequeue` 时里面没有元素，会把它从 `poolChain` 里删掉。
4. 从上一轮准备`GC` 的 `victim` 字段里尝试取对象。
5. 如果还没有，调用用户设置的 `New()` 函数，创建一个新的对象。





## 对象的清理

上面讲到，当窃取其他 `P` 的对象时，会逐步淘汰已经为空的 `poolDequeue`。但除此之外，`sync.Pool` 一定也还有其他的对象清理机制，否则对象池将可能会无限制的膨胀下去，造成内存泄漏。

`Golang` 对 `sync.Pool` 的清理逻辑非常简单粗暴。首先每个被使用的 `sync.Pool`，都会在初始化阶段被添加到全局变量 `allPools []*Pool` 对象中。`Golang` 的 `runtime` 会在 每轮 `GC` 前，触发调用 `poolCleanup()` 函数，清理 `allPools`。

```go
func poolCleanup() {
	for _, p := range oldPools {      // 清理老的对象池
		p.victim = nil
		p.victimSize = 0
	}

	for _, p := range allPools {      // 将 allPools 中所有的对象搬迁到 victim 字段
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	oldPools, allPools = allPools, nil // 将 allPools 复制给 oldPools，下一轮清理
    // 而 allPools 变成空的了，下次 Put 时会把在用的对象放回来
}
```

在每轮 `sync.Pool` 的清理中，暂时不会完全清理对象池，而是将其放在 `victim` 中。等到下一轮清理，才完全清理掉 `victim`。也就是说，每轮 `GC` 后 `sync.Pool` 的对象池都会转移到 `victim` 中，同时将上一轮的 `victim` 清空掉。

这么做是为了防止 `GC` 之后 `sync.Pool` 被突然清空，对程序性能造成影响。因此先利用 `victim` 作为过渡，如果在本轮的对象池中实在取不到数据，也可以从 `victim` 中取，这样程序性能会更加平滑。





# 总结

1. 利用 `GMP` 的特性，为每个 `P` 创建了一个本地对象池 `poolLocal`，尽量减少并发冲突。
2. 二级缓存：每个 `poolLocal` 都有一个 `private` 对象，优先存取 `private` 对象。
3. 如果 `private` 取不到，利用对象窃取的机制，从其他 `P` 的本地对象池以及 `victim` 中获取对象。






#### 参考

> [cyhone - 深度分析 Golang sync.Pool 底层原理](https://www.cyhone.com/articles/think-in-sync-pool)	