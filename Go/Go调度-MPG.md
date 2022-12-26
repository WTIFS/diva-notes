# 调度

所谓并发，是指 CPU 一会儿处理几下 A 程序的代码，一会儿又处理几下 B 的，看起来就好像一秒内同时在处理 A 和 B 程序。

而公平的分配 A 和 B 的运行时间，就是 **调度** 所做的事情。





# Go 调度器的发展历程

- 单线程调度器 0.x
  - 程序中只能存在一个活跃线程，由 G-M 模型组成
- 多线程调度器 1.0
  - 允许运行多线程程序
  - 全局锁导致竞争严重
- 任务窃取调度器 1.1
  - 引入了上下文P，构成了目前的 **M-P-G** 模型
  - 实现了基于**工作窃取**的调度器
  - 在某些情况下，Goroutine 不会让出线程，导致饥饿问题
  - 时间过长的垃圾回收 (Stop the world, STW) 会导致程序长时间无法工作
- 抢占式调度器 1.2 ~ 1.14
  - 基于**协作**的抢占式调度器 1.2 ~ 1.13
    - 通过编译器，在编译时向函数头部写入**抢占检查**代码，在函数调用时检查当前 `goroutine` 是否发起了抢占请求
    - `goroutine` 可能会因为垃圾回收和循环长时间占用资源导致程度卡住
  - 基于**信号**的抢占式调度器 1.14
    - 实现基于信号的真·抢占式调度
    - 垃圾回收在扫描栈时会触发抢占调度
    - 抢占的时间点不够多，还不能覆盖全部的边缘情况



# M，P 和 G

Go 的调度器使用三个结构体来实现 `goroutine` 的调度：`G, M, P`。

**G**：代表一个 `goroutine`。不过 `G` 并非执行体，它仅保存并发任务的状态，用自己独立的栈存放当前的运行内存及状态。

- 当 `G` 被调离 CPU 时，调度器代码负责把 CPU 寄存器的值保存在 `G` 对象的成员变量之中。当 `G` 被调度起来运行时，调度器代码又负责把 `G` 对象的成员变量所保存的寄存器的值恢复到 CPU 的寄存器。`G` 创建后被放置在 `P` 本地队列或全局队列，等待工作线程调度执行。

**M**：表示系统线程。它本身与一个内核线程进行绑定，是实际的执行体。

- 运行时还需要和一个 `P` 绑定，以循环调度方式不断从 `P` 里取 `G` 并执行。

**P**：代表一个虚拟的处理器，作用类似 `CPU` 核，用来控制可同时并发执行的任务数。

- 每个工作线程都必须绑定一个 `P` 才被允许执行任务，否则只能休眠，直到有空闲 `P` 时被唤醒。它还为线程执行提供资源，如内存 (`mcache`)、`G` 队列 (`local runq`) 、辅助GC等。线程独享绑定的 `P` 资源，可在无锁状态下执行高效操作。

除了上面三个结构体以外，还有一个存放所有可运行 `goroutine` 的容器 `schedt`。每个 `Go` 程序中 `schedt` 结构体只有一个实例对象，在代码中是一个共享的全局变量，每个工作线程都可以访问它以及它所拥有的 `goroutine` 运行队列。

另外，尽管 `P/M` 构成执行组合体，但两者数量并非一一对应。 `P` 的数量默认为 `CPU` 数，而 `M` 是由调度器按需创建的。比如，当 `M` 因陷入系统调用阻塞时，`P` 就会被监控线程抢回，去新建/唤醒一个 `M` 执行其他任务，这样 `M` 的数量就会增长。





# 初代调度模型 MG

一开始的调度模型只有 `G` 和 `M` 两个结构。

所有的 `G` 都放在一个全局队列 `global runqueue` 里，`M` 不断从这个队列里取 `G` 并执行 `G` 里保存的任务。

由于多个 `M` 都会从这个全局队列里获取 `G` 来进行运行，所以必须对该队列加锁保证互斥。这必然导致多个 `M` 对锁的竞争，这也是老调度器最大的一个缺点。

其实老调度器有4个缺点：详见 [Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw)

1. 创建、销毁、调度 `G` 都需要每个 `M` 获取锁，这就形成了激烈的锁竞争
2. `M` 转移 `G` 会造成延迟和额外的系统负载。比如在 `G` 中创建新协程的时候，`M` 创建了 `G2`，为了继续执行`G`，需要把 `G2` 交给 `M2` 执行，也造成了很差的局部性，因为 `G2` 和 `G` 是相关的，最近访问的数据都在 `M` 上，最好放在 `M` 上执行，而不是其他 `M2`。
3. `M` 中的 `mcache` 是用来存放小对象的，`mcache` 、栈都和 `M` 关联造成了大量的内存开销和很差的局部性
4. 系统调用导致频繁的线程阻塞和取消阻塞操作增加了系统开销。





# 次代调度模型 MPG

在 Dmitry Vyokov 实现的次代调度模型里，加上了 `P` 这一层，将全局唯一的一个 `G` 队列打散到了 `P` 里，每个 `P` 自己维护一个 `G` 的队列，解决原来的全局锁问题。

原来是一把大锁，锁定了全局协程队列这样一个共享资源，导致了很多激烈的锁竞争。如果我们要优化，很自然的就能想到，把锁的粒度细化来提高减少锁竞争，也就是把全局队列拆成局部队列。





# 详情

## G (goroutine) 
```go
// runtime/runtime2.go
type g struct {
    stack        stack      // 当前 Goroutine 的栈内存范围 [stack.lo, stack.hi) 
    stackguard0  uintptr    // 一个警戒指针，用来判断栈容量是否需要扩张
    m            *m         // 当前 Goroutine 绑定的 M
    sched        gobuf      // 调度相关的数据，上下文切换时就更新这个
    preempt      bool       // 抢占信号，标记 G 是否应该停下来被调度，让给别的 G
    timer        *timer     // 给 time.Sleep 用的 timer
    ...
    _panic       *_panic    // panic链表
    _defer       *_defer    // defer链表
}
```
- `G` 维护了 `goroutine` 需要的栈、程序计数器以及它所在的 `M` 等信息。
- `G` 并非执行体，**仅保存并发任务状态**，为任务提供所需要的栈空间
- `G` 创建成功后放在 `P` 的本地队列或者全局队列，等待被调度
- `G` 使用完毕后会被放回缓存池里，等待被复用
    - 因此存在：如果瞬发新建了大量 `G`，高峰后这些 `G` 会一直留着，不释放内存的问题。
    - 两级缓存池：每个 `P` 里有本地 `G` 缓存，`schedt` 里还有个全局 `G` 缓存，共两级
    - `P` 里的缓存池是 `gFree` 字段， `G` 工作结束后（状态变为 `Gdead`），会放入这里。读写它不需要锁
    - `P.gFree` 里的空闲 `G` 超过 `64` 个后，会分一半的 `G` 到 `schedt.gFree` 里。`schedt.gFree` 又根据 `G` 的栈大小分成了两个队列
      - 为节省空间，`G` 的栈如果超过 `2KB`，那回收时不会保留它的栈，放入 `schedt.gFree.noStack` 里
      - 否则放入另一个队列 `schedt.gFree.Stack` 里



##### sudog

当 `G` 遇到阻塞场景时，比如向 `channel` 发送/接收阻塞时，会被封装为 `sudog` 这样一个结构。其实就是给 `G` 加了个 `elem` 字段，用于记录 `channel` 发送的数据。

```go
type sudog struct {
    g     *g               // 正常的 G
    elem  unsafe.Pointer   // 保存数据的指针
}
```





## M (Machine) 线程
```go
type m struct {
    g0         *g     // 预留用于执行调度指令的 G。调度和执行 syscall 时会切换到这个 G。通过它, M 可以不通过 P 执行原生代码

    curg              // 当前运行的 G
    preemptoff string // 是否保持 curg 始终在这个 M 上运行
  
    p             puintptr // 当前挂的 P 
    nextp         puintptr // 下一个 P
    oldp          puintptr // 上一个运行的 P
  
    locks      int32  // 表示该 M 是否被锁，M 被锁的状态下该 M 无法执行 GC
    spinning   bool   // 是否自旋，自旋就表示 M 正在找 G 来运行。自旋状态的 M = 正在找工作 = 空闲的 M
    blocked    bool   // M 是否被阻塞
    ...
}
```
- 一个 `M` 就是一个用户级线程
- `M` 是一个很大的结构，里面维护小对象内存 `cache`、当前执行的 `goroutine`、随机数发生器等等非常多的信息。
- `M` 的数量目前最多10000个
- `M` 通过修改寄存器，将执行栈指向 `G` 自带的栈内存，并在此空间内分配堆栈帧，执行任务函数
- 中途切换时，将寄存器值保存回 `G` 的 `sched` 字段即可记录状态，任何 `M` 都可以通过读这个 `G` 的该字段来恢复该 `G `的状态。线程 `M` 本身只负责执行，不保存状态。这是并发任务跨线程调度，实现多路复用的根本所在。



`M` 并没有像 `G` 和 `P` 一样的状态标记，但可以认为一个 `M` 有以下的状态:

1. 自旋中 `spinning`： `M` 正在从运行队列获取`G`, 这时候 `M` 会拥有一个 `P`
2. 执行 `go` 代码中：  `M` 正在执行 `go` 代码, 这时候 `M` 会拥有一个 `P`
3. 执行原生代码中： `M` 正在执行阻塞的 `syscall` 或者 `cgo`, 这时 `M` 并不拥有 `P`（`M` 可以无 `P` 执行原生代码，运行在 `g0` 上）
4. 休眠中： `M` 发现没有待运行的 `G` 时会进入休眠, 并添加到空闲 `M` 链表中, 这时 `M` 并不拥有 `P`

自旋中 `spinning` 这个状态非常重要, 是否需要唤醒或者创建新的 `M` 取决于当前自旋中的 `M` 的数量。

通常创建一个 `M ` 的原因是由于没有足够的 `M` 来关联 `P` 并运行其队列里的 `G`。此外运行时 `sysmon` 时，或者 `GC` 的时候也会创建 `M`。





## P (Processor)，处理器
```go
// runtime.runtime2.go
type p struct {
    m           muintptr    // 挂载的 M
     
    mcache      *mcache     // 用于内存分配的结构体
  
    runq     [256]guintptr  // 本地可运行的 G 队列，local runqueue。长度固定256。通过队尾-队头判断实际长度
    runqhead uint32         // 队头
    runqtail uint32         // 队尾
    runnext guintptr        // 下一个可执行 G，优先级比 runq 队列高
    
    gFree struct {          // Gdead 状态的 G 池。新建 G 时，优先从这里取进行复用，其次再用 schedt 里的全局队列
        gList
        n int32
    }

    sudogcache []*sudog     // sudog 缓存池。二级缓存，优先从这里复用，再用 schedt 里的
    sudogbuf   [128]*sudog
}

var (
    allp []*p               // 全局 P 数组
)
```
- `P` 是线程 `M` 和 `G` 的中间层，用于调度 `G` 在 `M` 上执行。
- `P` 自身是用一个全局数组 `allp` 来保存的，长度默认为 `GOMAXPROCS`
- 它的主要用途就是用来执行 `goroutine` 的，所以它维护了一个本地 `G` 队列，里面存储了所有需要它来执行的 `goroutine`。
  - 本地队列避免了竞争锁（解决了初代模型最大的问题）。
- `P` 为线程提供了执行资源，如对象分配内存，本地 `G` 队列等。线程独享 `P` 资源，可以无锁操作
- 为何要维护多个 `P` 呢？是为了当一个 OS线程 `M` 陷入阻塞时，`P` 能从之前的 `M` 上脱离，转而在另一个 `M` 上运行
- `P` 控制了程序的并行度。可以通过 `GOMAXPROCS` 设置 `P` 的个数，最多只能有 `GOMAXPROCS` 个线程能并行，一般设为 CPU 核数





## Schedt

- `Schedt` 结构就是一个调度器，它维护有存储 `M` 和 `G ` 的队列以及调度器的一些状态信息等。
- 它是一个共享的全局变量，全局只有一个 `schedt` 类型的实例。
- 里面存放了空闲（即休眠） `M` 列表、空闲 `P` 列表、全局 `G` 队列、废弃的 `G` 队列

```go
// runtime/runtime2.go
type schedt struct {
    
    lock mutex 
     
    midle        muintptr  // 空闲的 M 列表
    nmidle       int32     // 空闲的 M 列表数量
    mnext        int64     // 下一个被创建的 M 的 id
    maxmcount    int32     // M 的最大数量，初始化时设为 10000，即最多 10000 个 M
    nmspinning uint32      // 处于自旋状态的 M 的数量

    pidle      puintptr    // 空闲 P 列表
    npidle     uint32      // 空闲 P 数量
    
    runq     gQueue        // 全局 runnable G 队列
    runqsize int32         // 全局 G 队列的长度
   
    gFree struct {         // 有效 dead G 的全局缓存。二级设计，新建 goroutine 时，优先从 P 复用，再从这里
        lock    mutex
        stack   gList      // 包含栈的
        noStack gList      // 没有栈的
        n       int32
    }
    
    sudoglock  mutex      
    sudogcache *sudog      // 全局 sudog 缓存。也是个二级缓存，使用时先用 P 的，再用全局的
     
    deferlock mutex
    deferpool [5]*_defer   // defer 结构的池
}
```





# 实现

## 程序启动后 MPG 的创建

- 调度器初始化函数 `runtime.schedinit`
    - `schedinit` 函数主要会创建一批 `P`，数量默认为 `CPU` 数。如果用户设置了 `GOMAXPROCS` 环境变量，则 `P` 的数量为 `max(GOMAXPROCS, 256)`，也就是最多 256。这些 `P` 初始创建好后都放置在 `Sched ` 的 `pidle` 队列里
- 调用 `runtime.newproc` 创建出第一个 `goroutine`，这个 `goroutine` 将执行的函数是 `runtime.main`
    - 这第一个 `goroutine` 也就是所谓的主 `goroutine`。我们写的最简单的 `Go` 程序 `"hello, world"` 就是完全跑在这个 `goroutine` 里
- 主 `goroutine` 开始执行后，做的第一件事情是创建了一个新的内核线程 `M`: 系统监控 `sysmon`
    - `sysmon` 用来检测长时间（超过 10ms）运行的 `goroutine`，将其调度到全局队列
- 此外还会启动垃圾回收、运行用户代码的 `gouroutine`
- 程序启动后，会给每个逻辑 `CPU` 分配一个 `P`；同时，会给每个 `P` 分配一个 `M`，这些 `M` 由 OS scheduler 来调度。





## 创建 G
- 使用 `go func()` 关键字时，会调用 `newproc` 函数，创建新的 `goroutine`
    - 不一定是从新创建，会先尝试复用空闲的 `G`
    - 先从 `P` 的 `gfree` 字段取空闲 `G`，没有再从 `sched.gfree` 链表里转移一批空闲 `G` 到 `P` 里，再重试
    - 再没有，才会新建
- 新的 `goroutine` 会被加到本地队列里
  - 通过 `go` 关键字新建的协程，会被放到本地队列的头部（`runnext` 字段）
  - 调度时，会从队列里 `pop` 一个 `goroutine`，设置栈和 `instruction pointer`，开始执行这个 `goroutine`
- `P` 的本地队列长度超过 `256` 时，里面一半的 `G` 会被转移至全局队列
- 任务队列分为三级，分别是 `P.runnext`, `P.runq`, `Sched.runq`，很有些 CPU 多级缓存的意思。

```go
// 创建新的 G
func newproc(siz int32, fn *funcval) {
    gp := getg()        // 获取当前的 G 
    pc := getcallerpc() // 获取调用者的程序计数器 PC
  
    systemstack(func() {                         // 系统调用
        newg := newproc1(fn, argp, siz, gp, pc)  // 创建新的 G 结构体
        _p_ := getg().m.p.ptr()   
        runqput(_p_, newg, true)                 // 将 G 加入到 P 的运行队列
      
        if mainStarted {
            wakep()                              // 这里还会触发一次唤醒空闲的 M 执行空闲的 P
        }
    })
}
```



```go
// runtime/proc.go
// runqput 尝试将一个 G 放入本地队列. 本地队列如果满了，放入全局队列
// next = false 时，放入队尾
// next = true 时，直接写入 P 的 runnext 字段
// 只能由 owner P 执行
// 另外可以看到由于使用了 P, 整个函数是无锁的
func runqput(_p_ *p, gp *g, next bool) {
    if next {                                                  // 新建G时，next为true，G会直接写入 runnext 字段
        oldnext := _p_.runnext
        _p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) // 直接写入 runnext 字段
        gp = oldnext.ptr()                                     // 原 runnext 里的 G 后面会被踢到本地/全局队列
    }
    
    // 虽然 runq 是循环队列，但 h 和 t 的值并不会 %256，t 会始终自增，因此可以通过 t-h 判断队列长度。赋值时才取模
    h := atomic.LoadAcq(&_p_.runqhead)            // 队头
    t := _p_.runqtail                             // 队尾
    if t-h < uint32(len(_p_.runq)) {              // 本地队列未满，可写入
        _p_.runq[t%uint32(len(_p_.runq))].set(gp) // 插到队列尾部
        atomic.StoreRel(&_p_.runqtail, t+1)       // 更新队尾指针
        return
    }
    if runqputslow(_p_, gp, h, t) { // 如果本地队列满了，分一半到全局队列。因为需要加锁，所以函数叫 slow
        return
    }
}

// 移动本地队列的前半部分，到全局队列。因为需要加锁，所以函数叫 slow
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
    var batch [len(_p_.runq)/2 + 1]*g

    n := t - h
    n = n / 2
    for i := uint32(0); i < n; i++ {
        batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr() // 把本地队列的前半部分到 batch 数组
    }
    if !atomic.CasRel(&_p_.runqhead, h, h+n) {                 // 更新本地队列的头指针，即删除本地队列的前半部分
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



## 创建P

`P` 只在初始化的时候创建，数量为 `GOMAXPROCS`，一般建完就不会再变了，除非用户调用 `runtime.MAXGOPROCS` 调整 `P` 的数量。

不过官方注释说 `runtime.MAXGOPROCS` 在后续改进调度器后会被移除。



## 创建 M

在 `newproc` 里可以看到一个 `wakep()` 函数，作用是唤醒空闲 `M` 执行空闲 `P`，如果有空闲 `P` 但没有空闲 `M`，就会新建一个 `M`。

`M` 一旦被创建，就不再会被销毁了，最多是休眠，放入 `sched` 的空闲 `M` 列表里。

```go
// runtime/proc.go
// 唤醒一个空闲 M 执行空闲 P
func wakep() {
    if atomic.Load(&sched.npidle) == 0 {                                            // 如果没有空闲P就返回
        return
    }
    if atomic.Load(&sched.nmspinning) != 0 || !atomic.Cas(&sched.nmspinning, 0, 1) { // 设为自旋状态，防止并发唤醒
        return
    }
    startm(nil, true) // 找空闲 M，或者新建一个 M
}

// 获取空闲 M 或新建 M
func startm(_p_ *p, spinning bool) {
    nmp := mget()              // 获取空闲 M 
    if nmp == nil {        // 没有空闲 M，新建一个
        id := mReserveID()     // sched.mnext 字段记录了 M 的自增 ID；如果超出 sched.maxmcount(默认10000)，会 panic
        newm(fn, _p_, id)  // 新建 M
        return
    }
}

// 新建一个M
func newm(fn func(), _p_ *p, id int64) {
    mp := allocm(_p_, fn, id) // 创建 M 对象
    newm1(mp)
}

// 创建M对象、分配空间、初始化
func allocm(_p_ *p, fn func(), id int64) *m {
    mp := new(m)        // 创建M对象 
    mp.mstartfn = fn    // 设置启动函数
    mcommoninit(mp, id) // 初始化
  
    mp.g0 = malg(8192 * sys.StackGuardMultiplier) // 初始化g0
    mp.g0.m = mp
}

// 为M创建系统线程
func newm1(mp *m) {
    newosproc(mp) // 创建系统线程
}

// runtime.os_linux.go
// 这个函数根据具体 os 的实现是不一样的，linux 下用的是 clone()，macOS 下用的就是 pthread_create()
// 初始 func 是 mstart，mstart() 会触发调度 schedule()
func newosproc(mp *m) {
    ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
}
```

`M` 比较特别的地方是自带一个名为 `g0`，默认 `8KB` 栈内存的 `G` 属性对象。它的栈内存地址被传给 `clone` 函数，作为系统线程的默认堆栈空间（Linux下）。

通过 `g0`，`M` 可以无 `P` 执行 `runtime` 管理指令，比如 `systemstack` 这种执行方式。

在进程执行过程中，有两类代码需要运行。

- 一类是用户逻辑，直接使用当前 `G` 的栈内存
- 另一类是运行时管理指令，它并不便于直接在用户栈上执行，因为涉及到切换，需要保存用户栈、执行自己的逻辑、再恢复用户栈，要处理与用户逻辑现场有关的一大堆事务，很麻烦。

因此，当需要执行管理指令时，会将线程栈临时切换到 `g0`，与用户逻辑彻底隔离。





## 调度 schedule

`G` 是如何被轮换并执行的呢？

1. 总体思路是：依次从 `P` 的本地 `G` 队列、全局 `G` 队列取
2. 调度器每调度 `61` 次，从全局队列里取一次 `G`（以避免全局队列里有的 `G` 始终不被取出、饿死）
3. 调用 `runqget` 函数从 `P` 本地的运行队列中查找待执行的 `G`
4. 调用 `findrunnable` 函数从其他地方取 `G`（依次从本地队列、全局队列、netpoll、从其他 `P` 里偷窃取 `G`）
    - 再次从 `local runq` 获取 `G`
    - 去 `global runq` 获取（因为前面仅仅是1/61的概率）
    - 执行 `netpoll`，检查是否有 `IO` 就绪的 `G`
    - 如果还是没有，那么随机选择一个 `P`，偷其 `runqueue` 里的一半。
        - 这里的随机用到了一种质数算法，保证既随机，每个 `P` 又都能被访问到
    - 偷窃前会将 `M` 的自旋状态设为 `true`，偷窃后再改回去
    - 如果多次尝试偷 `P` 都失败了，`M` 会把 `P` 放回 `sched` 的 空闲 `P` 数组，自身休眠（放回`M`池子）
5. `wakep`, 另一种情况是，`M` 太忙了，如果 `P` 池子里有空闲的 `P`，会唤醒其他 `sleep` 状态的 `M` 一起干活。如果没有`sleep`状态的`M`，`runtime` 会新建一个 `M`。
6. `execute`，执行代码。



#### schedule

```go
// runtime/proc.go
// 找出一个 G 来执行
func schedule() {

// STW检查
top:
    if sched.gcwaiting != 0 {
        gcstopm()
        goto top
    }

    // 检查定时器
    checkTimers(pp, 0)
  
    // 进入 GC MarkWorker 工作模式
    if gp == nil && gcBlackenEnabled != 0 {
        gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
        if gp != nil {
            tryWakeP = true
        }
    }
  
    // 每 tick 61次，从全局队列里取 G
    if gp == nil {
        if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
            lock(&sched.lock)
            gp = globrunqget(_g_.m.p.ptr(), 1)
            unlock(&sched.lock)
        }
    }

    // 从本地队列里取 G，优先使用 P.runnext 字段
    if gp == nil {
        gp, inheritTime = runqget(_g_.m.p.ptr())
    }
    
    // 通过 findrunnable 函数从其他可能得地方寻找可执行 G
    if gp == nil {
        gp, inheritTime = findrunnable() // blocks until work is available
    }
    
  
    // 执行任务函数 
    execute(gp, inheritTime)
}
```



#### findrunnable

依次从 本地队列、全局队列、netpoll、其他 `P` 获取可运行的 `G`。偷窃优先级最低，因为会影响其他 `P` 执行（需要加锁）。

```go
// 寻找可执行的 G
// Tries to steal from other P's, get g from local or global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
    now, pollUntil, _ := checkTimers(_p_, 0) // 检查 P 中的定时器，参考 Timer 的实现那篇文章
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
  
    // 网络协程
    if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
        // 略
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
```



#### globrunqget

在检查全局队列时，除返回一个可用 `G` 外，还会批量转移一批 `G` 到 `P` 的本地队列。毕竟不能每次加锁去操作全局队列。

```go
// 从全局队列列取 G
// 另外还会分一部分 G 给 P（分的 G 个数为 Min（全局队列长度/P个，或者P本地队列/2））
func globrunqget(_p_ *p, max int32) *g {
    if sched.runqsize == 0 {
        return nil
    }

    // 分给 P 的 G 个数为（全局队列长度/P个数，P本地队列长度/2，二者最小值）
    // 将全局队列按 P 个数等分
    n := sched.runqsize/gomaxprocs + 1
    if n > sched.runqsize {
        n = sched.runqsize
    }
    // 不能超过 runq 数组长度的一半
    if n > int32(len(_p_.runq))/2 {
        n = int32(len(_p_.runq)) / 2
    }

    sched.runqsize -= n    // 调整计数

    gp := sched.runq.pop() // 从全局队列头部弹出 G
    n--
    for ; n > 0; n-- {     // 分 n 个 G 到当前 P 的本地队列
        gp1 := sched.runq.pop()
        runqput(_p_, gp1, false)
    }
    return gp
}
```





## 调度时机

- 新建 `M` 后，`mstart` 里触发 `schedule`
- 新建 `G` 后（`go func`）
- 阻塞性系统调用，比如文件 IO，网络IO
  - `Golang` 重写了所有系统调用，在系统调用里加入了调度逻辑（更改 `G`状态、`M` 和`P` 解绑、 `schedule `等）
- `sysmon` 会定期检查运行超长的任务，进行调度
- 主动调用 `runtime.Gosched()`，这个很少见，一般调试才用





# 优点

- 调度是在用户态下完成的， 不涉及内核态与用户态之间的频繁切换
- 不需要频繁的进行内存的分配与释放。在用户态维护着一块大的内存池， 不直接调用系统的 `malloc` 函数（除非内存池需要改变）
- 另一方面充分利用了多核的硬件资源，近似的把若干 `goroutine` 均分在物理线程上
- 再加上本身 `goroutine` 的超轻量，以上种种保证了 `go` 调度方面的性能





## 问题

##### P 的 runq大小为什么是 256？

太小会频繁触发和全局队列的交换。太大会频繁触发偷窃。





#### 参考

> [Golang调度与MPG](https://www.jianshu.com/p/af80342a1233)
>
> [深入理解Go语言(03)：scheduler调度器 - 基本介绍](https://www.cnblogs.com/jiujuan/p/12735559.html)
>
> [golang 调度学习-综述](https://blog.csdn.net/diaosssss/article/details/92830782)
>
> [golang核心原理-协程调度时机](https://studygolang.com/articles/34362)
>
> [lucifer_L49715 - 理解golang调度之二 ：Go调度器](https://juejin.cn/post/6844903846825705485)

