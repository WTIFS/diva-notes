# 初代调度模型 MG
`go` 的调度模型，一开始并不是 `MPG` 模型，而是经过了一个发展过程。老的调度模型只有 `G` 和 `M`，没有 `P`。
它还有一个重要的数据结构：全局队列 `global runqueue`。全局队列是用来存放 `goroutine` 的。启动那么多 `goroutine`，总要一个地方把 `G` 存起来以便 `M` 来调用。

多个 `M` 会从这个全局队列里获取 `G` 来进行运行。所以必须对全局队列加锁保证互斥。这必然导致多个 `M` 对锁的竞争。这也是老调度器的一个缺点。

其实老调度器有4个缺点：详见 [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)

1. 创建、销毁、调度 `G` 都需要每个 `M` 获取锁，这就形成了激烈的锁竞争
2. `M` 转移 `G` 会造成延迟和额外的系统负载。比如在 `G` 中创建新协程的时候，`M` 创建了 `G2`，为了继续执行`G`，需要把 `G2` 交给 `M2` 执行，也造成了很差的局部性，因为 `G2` 和 `G` 是相关的，最近访问的数据都在 `M` 上，最好放在 `M` 上执行，而不是其他 `M2`。
3. `M` 中的 `mcache` 是用来存放小对象的，`mcache` 、栈都和 `M` 关联造成了大量的内存开销和很差的局部性
4. 系统调用导致频繁的线程阻塞和取消阻塞操作增加了系统开销。



# 次代调度模型 MPG

在大神 Dmitry Vyokov 实现的次代调度模型里，加上了 `P` 结构体。每个 `P` 自己维护一个处于 `Runnable` 状态的 `G` 的队列，解决了原来的全局锁问题。

原来是一把大锁，锁定了全局协程队列这样一个共享资源，导致了很多激烈的锁竞争。如果我们要优化，很自然的就能想到，把锁的粒度细化来提高减少锁竞争，也就是把全局队列拆成局部队列。


支撑整个调度器的主要有4个重要结构，分别是`M、G、P、Schedt(schedule)`，前三个定义在 `runtime.h` 中，`Sched` 定义在 `proc.c` 中。



## Schedt

- `Schedt` 结构就是调度器，它维护有存储`M`和`G`的队列以及调度器的一些状态信息等。
- 全局调度器，全局只有一个 schedt 类型的实例:



## M (Machine) 内核级线程

- 一个 `M` 就是一个用户级线程；`M` 是一个很大的结构，里面维护小对象内存 `cache`、当前执行的`goroutine`、随机数发生器等等非常多的信息。
- `M` 的数量目前最多10000个
- `M` 通过修改寄存器，将执行栈指向 `G` 自带的栈内存，并在此空间内分配堆栈帧，执行任务函数
- 中途切换时，将寄存器值保存回 `G` 的空间即可维持状态，任何 `M` 都可以恢复这个 `G`。线程只负责执行，不保存状态，这是并发任务跨线程调度，实现多路复用的根本所在

```go
type m struct {
    g0         *g     // 用来执行调度指令的 goroutine，调度和执行系统调用时会切换到这个 G，通过这个g0, M 可以无 P 执行原生代码
    curg              // 当前运行的g
    preemptoff string // 该字段不等于空字符串的话，要保持 curg 始终在这个 m 上运行
    locks      int32  // locks表示该M是否被锁的状态，M被锁的状态下该M无法执行gc
    spinning   bool   // 是否自旋，自旋就表示M正在找G来运行。自旋状态的 `M` = 正在找工作 = 空闲的 `M`
    blocked    bool   // m是否被阻塞
}
```

`M` 并没有像 `G` 和 `P` 一样的状态标记, 但可以认为一个 `M` 有以下的状态:
1. 自旋中 `spinning`: `M` 正在从运行队列获取`G`, 这时候 `M` 会拥有一个 `P`
2. 执行 `go` 代码中:  `M` 正在执行 `go` 代码, 这时候 `M` 会拥有一个 `P`
3. 执行原生代码中: `M` 正在执行原生代码或者阻塞的 `syscall`, 这时 `M` 并不拥有 `P`（`M` 可以无 `P` 执行原生代码，运行在 `g0` 上）
4. 休眠中: `M` 发现没有待运行的 `G` 时会进入休眠, 并添加到空闲 `M` 链表中, 这时 `M` 并不拥有 `P`

自旋中 `spinning` 这个状态非常重要, 是否需要唤醒或者创建新的 `M` 取决于当前自旋中的 `M` 的数量。

通常创建一个 `M `的原因是由于没有足够的 `M` 来关联 `P` 并运行其队列里的 `G`。此外运行时系统执行系统监控的时候，或者 `GC` 的时候也会创建 `M`。



## P (Processor)，处理器
- 它的主要用途就是用来执行 `goroutine` 的，所以它也维护了一个 `goroutine` 队列 `run queue`，即常说的本地队列，里面存储了所有需要它来执行的 `goroutine`。
- `P` 是用一个全局数组 (255) 来保存的，并且维护着一个全局的空闲 `P` 链表
- 为了避免竞争锁（解决初代模型最大的问题），每个 `P` 都有自己的本地队列。
- `P` 为线程提供了执行资源，如对象分配内存，本地任务队列等。线程独享 `P` 资源，可以无锁操作
- 为何要维护多个 `P` 呢？是为了当一个 OS线程 `M` 陷入阻塞时，`P` 能从之前的 `M` 上脱离，转而在另一个 `M` 上运行
- `P` 控制了程序的并行度，如果只有一个 `P`，那么同时只能执行一个任务，可以通过 `GOMAXPROCS` 设置 `P` 的个数



## G (goroutine) 
- `G` 维护了 `goroutine` 需要的栈、程序计数器以及它所在的 `M` 等信息。
- `G` 并非执行体，它仅仅保存并发任务状态，为任务提供所需要的栈空间
- `G` 创建成功后放在 `P` 的本地队列或者全局队列，等待调度

```go
type g struct {
    preempt       bool       //抢占信号，标记G是否应该停下来被调度，让给别的G
	timer          *timer    // cached timer for time.Sleep
}
```

#### sudog

当 `G` 遇到阻塞，或需要等待的场景时，会被打包成 `sudog` 这样一个结构。一个 `G` 可能被打包为多个 `sudog` 分别挂在不同的等待队列上。
`sudog` 代表在等待列表里的 `G`，比如向 `channel` 发送/接收内容时




# 关系

地鼠 (gopher) 用小车运着一堆待加工的砖。`M` 可以看作的地鼠，`P` 就是小车，`G` 就是小车里装的砖。



#### 抛弃P
你可能会想，为什么一定需要一个上下文 `P`，我们能不能去掉上下文，让 `runqueues` 直接挂到 `M` 上呢？答案是不行，需要 `P` 的目的，是为了在内核线程阻塞的时候，可以直接放开其他线程。

一个很简单的例子就是系统调用 `sysall`，一个线程肯定不能同时执行代码和系统调用被阻塞，这个时候，此线程 `M` 需要放弃当前的上下文环境 `P`，让其余 `M` 继续执行 `P` 中的其他 `G`。以便可以让其他的`Goroutine` 被调度执行。



# 调度实现
## 程序启动
- 调度器初始化 `runtime.schedinit`
    - `schedinit` 函数主要根据用户设置的 `GOMAXPROCS` 值来创建一批 `P`，不管 `GOMAXPROCS` 设置为多大，最多也只能创建 256 个 `P`。这些 `P` 初始创建好后都放置在 `Sched ` 的 `pidle` 队列里
- 调用 `runtime.newproc` 创建出第一个 `goroutine`，这个 `goroutine` 将执行的函数是 `runtime.main`
- 这第一个 `goroutine` 也就是所谓的主 `goroutine`。我们写的最简单的 `Go` 程序 "hello, world" 就是完全跑在这个 `goroutine` 里
- 主 `goroutine` 开始执行后，做的第一件事情是创建了一个新的内核线程 `M`: 系统监控 `sysmon`
- `sysmon` 用来检测长时间（超过 10 ms）运行的 `goroutine`，将其调度到全局队列。这个队列的优先级比较低
- 此外还会启动垃圾回收、运行用户代码的 `gouroutine`
- go 程序启动后，会给每个逻辑核心分配一个 `P`；同时，会给每个 `P` 分配一个 `M`，这些内核线程仍然由 OS scheduler 来调度。



## 创建 goroutine
- 使用 `go func()` 关键字时，会调用 `newproc`，创建新的 `goroutine`
- 新的 `goroutine` 会被加到本地队列里
  - 通过 `go` 关键字新建的协程，会被放到本地队列的头部
  - 到了调度的点后，会从 `runqueue pop` 一个 `goroutine`，设置栈和 `instruction pointer`，开始执行这个 `goroutine`
- `P` 的本地队列长度超过 64 时，里面一半的 `G` 会被转移至全局队列
- `findrunnable`，见下

```go
// runtime/proc.go
// runqput 尝试将一个 G 放入本地队列. 本地队列如果满了，放入全局队列
// next = false 时，放入队尾
// next = true 时，直接写入 P 的 runnext 字段
// 只能由 owner P 执行
// 另外可以看到由于使用了 P, 整个函数是无锁的
func runqput(_p_ *p, gp *g, next bool) {
    if next {
        oldnext := _p_.runnext
        _p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) //直接写入 runnext 字段
        gp = oldnext.ptr()                                     //原 runnext 里的 G 后面会被踢到全局队列
    }
    
    h := atomic.LoadAcq(&_p_.runqhead)            // 队头
    t := _p_.runqtail                             // 队尾
    if t-h < uint32(len(_p_.runq)) {              // 本地队列未满，可写入
        _p_.runq[t%uint32(len(_p_.runq))].set(gp) // 插到队列尾部
        atomic.StoreRel(&_p_.runqtail, t+1)       // 更新队尾指针
        return
    }
    if runqputslow(_p_, gp, h, t) { // 如果本地队列满了，分一半到全局队列
        return
    }
}

// 移动本地队列的前半部分，到全局队列
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
    var batch [len(_p_.runq)/2 + 1]*g

    // First, grab a batch from local queue.
    n := t - h
    n = n / 2
    for i := uint32(0); i < n; i++ {
        batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr() // 把本地队列的前半部分到 batch 数组
    }
    if !atomic.CasRel(&_p_.runqhead, h, h+n) { // 更新本地队列的头指针，即删除本地队列的前半部分
        return false
    }
    
    // 把 batch 数组做成链表，这样后面放入全局队列时只需要把 batch 头结点接到全局队列尾节点后面就可以了
    for i := uint32(0); i < n; i++ {
        batch[i].schedlink.set(batch[i+1]) 
    }
    
    var q gQueue
    q.head.set(batch[0]) 
    q.tail.set(batch[n])
    lock(&sched.lock)                // 操作全局队列，要加锁了
    globrunqputbatch(&q, int32(n+1)) // 把 batch 数组放到全局队列尾部
    unlock(&sched.lock)
}
```





## 调度 schedule()

1. 调度器每调度61次，从全局队列里取一次 `G`，以避免饥饿
2. `runqget`, `M`试图从自己的 `P` 的本地队列里取出一个任务`G`
3. `findrunnable` ，依次从本地、全局、netpoll、偷窃里取 `G`
    - 再次从 `local runq` 获取 `G`
    - 去 `global runq` 获取（因为前面仅仅是1/61的概率）
    - 执行 `netpoll`，检查是否有 `io`就绪的 `G`
    - 如果还是没有，那么随机选择一个P，偷其 `runqueue` 里的一半。（这里的随机用到了一种质数算法，保证既随机，每个P又都能被访问到）
    - 偷窃前会将 `M` 的自旋状态设为 `true`，偷窃后再改回去
    - 如果多次尝试偷 `P` 都失败了，`M` 会把 `P` 放回 `sched` 的 空闲 `P` 数组，自身 `sleep`（放回`M`池子）

4. `wakep`, 另一种情况是，`M`太忙了，如果`P`池子里有空闲的`P`，会唤醒其他`sleep`状态的`M`一起干活。如果没有`sleep`状态的`M`，`runtime`会新建一个`M`。

5. `execute`，执行代码。

```go
// runtime/proc.go
// 找出一个 G 来执行
func schedule() {

    // 每 tick 61次，从全局队列里取 G
    if gp == nil {
        // Check the global runnable queue once in a while to ensure fairness.
        // Otherwise two goroutines can completely occupy the local runqueue
        // by constantly respawning each other.
        if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock)
            gp = globrunqget(_g_.m.p.ptr(), 1)
            unlock(&sched.lock)
        }
    }

    // 从本地队列里取 G
    if gp == nil {
        gp, inheritTime = runqget(_g_.m.p.ptr())
    }
    
    // 通过 findrunnable 寻找可执行 G
    if gp == nil {
        gp, inheritTime = findrunnable() // blocks until work is available
    }
    
    // Poll network. 这一步只是优化，跳过影响也不大
    if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
        // 跳过
    }

    // 从其他 P 偷窃
    for i := 0; i < 4; i++ {
        // 这里的随机用到了一种质数算法，保证既随机，每个P又都能被访问到
        for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
            stealRunNextG := i > 2 // first look for ready queues with more than 1 g
            p2 := allp[enum.position()]
            if gp := runqsteal(_p_, p2, stealRunNextG); gp != nil {
                return gp, false
            }
        }
    }   
}

// 寻找可执行的 G
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
    // 本地队列
    if gp, inheritTime := runqget(_p_); gp != nil {
        return gp, inheritTime
    }

    // 本地队列里没G，从全局队列里取
    if sched.runqsize != 0 {
        lock(&sched.lock)          // 全局队列需要加锁        
        gp := globrunqget(_p_, 0)  // 从全局队列队头弹出；另外会分一部分 G 给当前 P
        unlock(&sched.lock)
        if gp != nil {
            return gp, false
        }
    }

}

// 从全局队列列取 G
// 另外还会分一部分 G 给 P
func globrunqget(_p_ *p, max int32) *g {
    if sched.runqsize == 0 {
        return nil
    }

    // n取 全局队列长度 / P本地队列长度/2 的最小值
    n := sched.runqsize/gomaxprocs + 1
    if n > sched.runqsize {
        n = sched.runqsize
    }
    if n > int32(len(_p_.runq))/2 {
        n = int32(len(_p_.runq)) / 2
    }

    sched.runqsize -= n

    gp := sched.runq.pop() // 从全局队列头部弹出 G
    n--
    for ; n > 0; n-- {     // 分一部分 G 到当前P的本地队列
        gp1 := sched.runq.pop()
        runqput(_p_, gp1, false)
    }
    return gp
}
```





## 线程阻塞

- 在同一时刻，一个线程上只能跑一个 `goroutine`。当 `goroutine` 发生阻塞（例如向一个五环充 `channel` 发送数据）时，`runtime` 会把当前 `goroutine` 调度走，让其他 `goroutine` 来执行。目的就是不让一个线程闲着，榨干 CPU 的每一滴油水。

- 当一个 `M` 陷入阻塞、阻塞结束后返回时，它会尝试取一个 `P` 来继续执行后面的。如果没有可用的 `P`，那么它会把 `G` 放到全局队列里，然后自身睡眠。



# 优点

- 调度是在用户态下完成的， 不涉及内核态与用户态之间的频繁切换
- 不需要频繁的进行内存的分配与释放。在用户态维护着一块大的内存池， 不直接调用系统的 `malloc` 函数（除非内存池需要改变）
- 另一方面充分利用了多核的硬件资源，近似的把若干 `goroutine` 均分在物理线程上
- 再加上本身 `goroutine` 的超轻量，以上种种保证了 `go` 调度方面的性能



# 参考

[The Go Scheduler](https://morsmachine.dk/go-scheduler)
[Golang调度与MPG](https://www.jianshu.com/p/af80342a1233)
[深入理解Go语言(03)：scheduler调度器 - 基本介绍](https://www.cnblogs.com/jiujuan/p/12735559.html)
[golang 调度学习-综述](https://blog.csdn.net/diaosssss/article/details/92830782)