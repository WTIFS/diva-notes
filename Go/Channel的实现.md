# 理念


> 不要通过共享内存来通信，而应该通过通信来共享内存

这句话是一条编程指导建议，和 golang 本身关系不大。



#### 共享内存

如果在一个系统中，两个线程或进程，都可以读写同一块内存空间，这就叫做"内存共享"。直觉上会觉得这种方式非常方便。

所有必要的信息都在内存中，想取就可以随时取。

但是会有数据冲突。为了对抗数据冲突，人们发明了很多机制。比如加锁、信号量、原子锁 (Automic)、巧妙的多线程算法等等。但是这些算法看起来很高级，实际上要么会 **影响并发性能、要么对使用场景有要求、要么很烧脑很难证明正确性**。

在实践中，运行效率再高，也得以正确性为前提才可以。如果因为并发影响了结果正确性，那就毫无意义；而如果因为照顾正确性影响了并发性能，那不如直接写成单线程程序。



#### 通信

人们通过多年试错和迭代，最终“通过通信来实现进程/线程间交互”的方案脱颖而出，成为了大多数人的共识。

通过通信让多线程/多进程交互，有多种具体的技术方案。erlang、go等语言在语言核心层提供了相应的功能，其它语言比如c#可以通过Task和相关的库提供类似功能。游戏服务器领域的skynet（用c语言+lua编写）也从零实现了高性能的actor框架。

各种技术方案背后的思想高度相似——先提供一个或多个高性能队列，线程/进程/微服务之间需要访问别人时，不能直接读写别人的数据，而要通过队列提出请求，然后在对方处理请求时再做相应处理。

- 这种设计可能会增加代码的复杂度，但优点是系统有高正确性。所有事件整体看起来像是串行的，你不需要写完程序后花大量时间调试各种多线程竞争问题。

- 如果使用无锁队列，还可以减少锁导致的延迟，性能甚至可以比共享内存方案还快。在部分顶尖低延迟领域如高频交易领域，这种范式越来越主流。

 

# 源码

## channel结构

```go
//src/runtime/chan.go

type hchan struct {
	qcount   uint           // 当前队列里的元素个数
	dataqsiz uint           // 环形队列长度，即缓冲区的大小，即 make(chan T, N) 中的 N
	buf      unsafe.Pointer // 环形队列
	elemsize uint16         // 每个元素的大小
	closed   uint32         // 标识当前通道是否处于关闭状态，创建通道后，该字段设置0，即打开通道；通道调用close将其设置为1，通道关闭
	elemtype *_type         // 元素类型，用于数据传递过程中的赋值
	sendx    uint           // 环形队列尾指针
	recvx    uint           // 环形队列头指针

	recvq waitq             // 等待读消息的goroutine队列
	sendq waitq             // 等待写消息的goroutine队列

	lock mutex //互斥锁，为每个读写操作锁定通道，因为发送和接受必须是互斥操作
}
  
// sudog 代表 goroutine
type waitq struct {
        first *sudog //这个是链表，通过next指向下一个sudog
        last  *sudog //链表尾部
}
```



## 写入

1. 锁定整个通道结构
2. 如果 `recvq` 中有等待读的 `goroutine`，则直接将元素写入该 `goroutine` (取出G、写入G、唤醒G)
    1. 这步节省了一个锁和内存 `copy` 的步骤。一般共享内存是：G1加锁、G1写入堆、G1解锁；G2加锁、G2读堆、G2解锁
    2. 这里直接是：G1加锁、G1写入G2的栈、G1唤醒G2、G1解锁
3. 如果 `recvq` 为空，则确定缓冲区是否可用，如果可用那么从当前 `goroutine` 复制数据到 `buf` 缓冲区中，并增加 `sendx` 下标的值
4. 如果缓冲区已经满了，则
    1. 要写入的元素将保存在当前执行的 `goroutine` 结构中，let's say `G1`
    2. 调度器将 `G1` 的状态设置为 `waiting`，移除与线程 `M` 的联系。此时 `G1` 就是阻塞状态
    3. `G1` 变为 `waiting` 状态后，会创建一个代表自己的 `sudog` 的结构，将数据保存到 `sudog.elem` 字段，然后追加到 `sendq` 链表中
5. 写入完成释放锁

```go
// runtime/chan.go
// 注意这里的 block 并非表示 channel 是阻塞/非阻塞的，而是表示是否通过 select: case chan <- data 的方式来访问
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    if c == nil {                                      // select 模式下空 channel 直接返回；否则报错
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2) // 向 nil channel 写会直接 gopark，进入阻塞
		throw("unreachable")
	}
    
    if !block && c.closed == 0 && full(c) {            // 快速返回：select 模式下 channel 写满时直接返回 false
		return false
	}

	lock(&c.lock)

	if sg := c.recvq.dequeue(); sg != nil {            // 如果等待队列里有等待的读者，直接把数据写到它的栈里
		send(c, sg, ep, func() { unlock(&c.lock) }, 3) // send 里面使用了 memmove 将数据复制给接收者
		return true
	}

    // buffer 还有空间
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

    // buffer 没有空间了，非阻塞模式下直接返回
	if !block {
		unlock(&c.lock)
		return false
	}

	// 同步模式
	gp := getg()
	mysg := acquireSudog() // 复用或新建一个 sudog
	mysg.releasetime = 0

	mysg.elem = ep         // 保存发送的数据
	mysg.g = gp            
	mysg.c = c
	c.sendq.enqueue(mysg)  // 将 sudog 放入发送者队列
    
    // 陷入休眠
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2) 

    // 到这里，说明休眠结束，有读者把数据读走了
    
    KeepAlive(ep) // 里面其实是 println(ep)，用来保证读者读数据前，数据不被 GC 掉

    // 恢复 G 的状态，并将 sudog 放回缓存池
	gp.waiting = nil
	gp.activeStackChans = false
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)  // 将 sudog 放回缓存池，下次复用

	return true
}

func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep) // 使用了 memmove 将数据复制给接收者
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	goready(gp, skip+1)                // 唤醒 gopark 了的接收者
}
```





## 读取

1. 快速返回：先判断 `channel` 是否关闭或者为空，是则直接返回
2. 获取 `channel` 全局锁
3. 尝试从 `sendq` 等待队列中获取等待写的 `goroutine`
4. 如果有等待的 `goroutine`
    1. 没有缓冲区：取出 `goroutine` 并读取数据，然后唤醒这个 `goroutine`，结束读取释放锁
    2. 有缓冲区 (有缓冲区的情况下还有等待的 `goroutine`，说明缓冲区此时满了)：从缓冲区队列队头取数据，作为返回的值；再把刚刚 `sendq` 里取出的那个 `goroutine` 放到缓冲队列队尾
5. 如果没有等待的 `goroutine`
    1. 缓冲区有数据：直接读取缓冲区数据
    2. 没有缓冲区或者缓冲区为空，将当前 `goroutine` 加入到` sendq` 队列，进入睡眠，等待被写入 `goroutine` 唤醒
6. 这个过程其实有点像车站排队打的。出租车是待写入的消息，乘客是读取者，停车场是 `buffer`。如果出租车等不到乘客，或者乘客等不到车，都得排队等着（阻塞）。如果有 `buffer`，车就先去停车场里。



```go
// 读取的源码和写入其实大同小异
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    if c == nil {          // select 模式下空 channel 直接返回；否则报错
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
    
    if !block && empty(c) { // 快速返回：select 模式下 channel 为空时直接返回 false
        if atomic.Load(&c.closed) == 0 {
			return
		}
        if empty(c) {       // 还做了 double check
            return true, false
        }
    }
    
    //加锁
    lock(&c.lock)
    
    // channel 关闭了，返回一个类型默认的零值
    if c.closed != 0 && c.qcount == 0 { 
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep) // 返回 ep 类型默认的零值
		}
		return true, false
	}

    if sg := c.sendq.dequeue(); sg != nil {
        // 如果 sendq 中有等待写的 goroutine，则判断 buffer。
        //   如果 buffer 为空，直接从 sender 的栈中读数据
        //   否则从 buffer 头部读数据，将 sender 的数据放入 buffer 队尾
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
    
    //buffer里有数据，则从buffer读取
	if c.qcount > 0 {
		qp := chanbuf(c, c.recvx)
		c.recvx++
       //循环队列，下标满了则重置
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
    
    // 没有buffer了，将 g 放入 recvq 并阻塞
    // no sender available: block on this channel.
	mysg := acquireSudog()
  	c.recvq.enqueue(mysg)
}

// 从 sendq 里读数据
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // 如果 buffer 为空，直接从 sendq 里的 sg 复制元素到 ep
	if c.dataqsiz == 0 {
		if ep != nil {
			recvDirect(c.elemtype, sg, ep)
		}
    // 如果 buffer 不为空，那么 buffer 一定是满的
	} else {
		qp := chanbuf(c, c.recvx)             // 取出 buffer 的头部元素，复制到 ep
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemmove(c.elemtype, qp, sg.elem) // 再把 sg 的值复制回头部那个元素
		c.recvx++                             // 这里很 tricky，没有用弹出头部、再插入队尾的做法
		if c.recvx == c.dataqsiz {            // 而是直接原地更新头部元素值
			c.recvx = 0                       // 由于 buffer 是个环形队列，因此更新队尾偏移量就可以了
		}
		c.sendx = c.recvx                     // 也更新 buffer 的写入偏移量
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	goready(gp, skip+1)                       // 唤醒gopark 状态下的 sudog
}
```





## close

关闭操作将所有排队者唤醒，并设置 `closed`、`param` 字段。

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	c.closed = 1

	var glist gList

	// 释放所有接收者
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg) // 这个参数表名唤醒者是 closechan
		sg.success = false
		glist.push(gp)
	}

	// 释放所有发送者（发送者继续发送会panic）
	for {
		sg := c.sendq.dequeue()
        // 和上面一样
        ...
	}
	unlock(&c.lock)

	// 唤醒所有 G
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```





## select

对读写 `channel` 的 `<- chan`、`chan ->` 这种写法，编译器会翻译成对函数的调用。



#### 写

对于 `x := <- c` 这类的阻塞操作，会编译为 `chansend1(c, x)`。

对于 `select`，则编译为 `if selectnbsend(c, x) {} else {}` 这种逻辑。

两者底层都调用的 `chansend` 函数，但传的 `block`参数不同。阻塞操作传 `true`，`select` 传 `false`。

这样在使用 `select` 的写法时，管道才能不阻塞的立即返回 `false`，`case` 才能跳过这个 `false`，无阻塞的继续向下判断 `case2`、`case3`。

```go
// c <- x 
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

// select case c <- 写法
// 编译器会将
//	select {
//	case c <- v:
//		... foo
//	default:
//		... bar
//	}
//
// 编译为
//
//	if selectnbsend(c, v) {
//		... foo
//	} else {
//		... bar
//	}
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
```



#### 读：

```go
// x:= <- c 
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

// x, ok := <- c
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

// select case <- c 写法
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
	return chanrecv(c, elem, false)
}
```



#### select case

每条 `case` 会被包装成一个 `scase` 对象，里面包含了 `channel`，和数据元素。

```go
//runtime/select.go
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```

`select - case` 则会被编译为对 `selectgo` 函数的调用。里面将所有 `case` 组成了一个 `scase` 数组，以随机顺序遍历这个数组，如果能读写数据，就返回；否则新建一个 `sudog` 放到 `scase.c` 这个 `channel ` 的等待队列里，等待唤醒。如果所有 `scase` 里都没值，则最后执行 `default`。

```go
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) {
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))      // case 数组
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0)) // 顺序数组

    ncases := nsends + nrecvs                            // case 数量
	scases := cas1[:ncases:ncases]                       // case 数组
	pollorder := order1[:ncases:ncases]                  // 遍历顺序
	lockorder := order1[ncases:][:ncases:ncases]         // 加锁顺序
    
    norder := 0                                          
	for i := range scases {                 // 生成随机排列顺序
		cas := &scases[i]
		j := fastrandn(uint32(norder + 1))  // 居然是蓄水池抽样
		pollorder[norder] = pollorder[j]    // 以 1/i+1 的概率对调 j 和 i+1
		pollorder[j] = uint16(i)
		norder++
	}
    
    // 阶段1： 随机顺序遍历
    for _, casei := range pollorder { 
		casi = int(casei)              // 大无语命名
		cas = &scases[casi]
		c = cas.c

		if casi >= nsends {
			sg = c.sendq.dequeue()   // 从等待队列里取 sudog
			if sg != nil {           // 取到了，就执行同步读
				goto recv
			}
			if c.qcount > 0 {        // 否则判断 buffer
				goto bufrecv         // 有 buffer 则读 buffer
			}
			if c.closed != 0 {
				goto rclose
			}
		} else {                     // 和上面差不多
			...
		}
	}

	if !block {                      // select 里有 default 时，block = false，执行这里
		selunlock(scases, lockorder) // 解锁
		casi = -1                    // 返回值为 -1
		goto retc                    // 返回
	}

	// 阶段2：所有 case 都阻塞住了，且没有 default
    // 将 select 挂到所有 case 的等待列表里，等待被唤醒
	gp = getg()
	nextp = &gp.waiting
	for _, casei := range lockorder { 
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog()    // 新建 sudog
		sg.g = gp
		sg.isSelect = true
		sg.elem = cas.elem
		sg.c = c
		if casi < nsends {
			c.sendq.enqueue(sg) // 挂到写等待队列
		} else {
			c.recvq.enqueue(sg) // 挂到读等待队列
		}
	}
}
```







## 产生阻塞/panic的情况

以下均不考虑 `select: case <- chan` 的情况

1. 读空的 `channel`
2. 写阻塞 `channel` / 写入数据超过缓冲区 
3. 读写 `nil channel`（阻塞。主线程下会导致 `fatal error: all goroutines are asleep`）

| 操作        | nil channel | 正常 channel | 已关闭 channel |
| ----------- | ----------- | ------------ | -------------- |
| `<- ch`     | 阻塞        | 成功 / 阻塞  | 读到零值       |
| `ch <-`     | 阻塞        | 成功 / 阻塞  | `panic`        |
| `close(ch)` | `panic`     | 成功         | `panic`        |





## 样例

读空 `channel`，阻塞
```go
t := make(chan int)
x, ok := <- t //阻塞在这里。主线程被放进t的recvq了，然后主线程被挂起，等待t写入后唤醒主线程
```



读已关闭的 `channel`，正常执行

```go
t := make(chan int)
close(t)
x, ok := <- t //0, false
```



写阻塞 `channel`，阻塞

```go
t := make(chan int)
t <- 1 //阻塞在这里，必须有其他gorutine消费才可以继续执行
```







#### 参考

> [马遥 - 如何理解 Golang 中 "不要通过共享内存来通信，而应该通过通信来共享内存"](https://www.zhihu.com/question/58004055/answer/1972944380)
>
> [Go channel 实现原理分析](https://www.jianshu.com/p/d841f251d3bc)
>
> [Go channel 源码](https://github.com/golang/go/blob/go1.12.7/src/runtime/chan.go#L183)
