单核CPU，开两个 `goroutine`，其中一个死循环，会怎么样？

答案是：死循环的 `goroutine` 卡住了，但是完全不影响另一个 `goroutine` 和 `主goroutine` 的运行。

go1.14 版本实现了基于信号的抢占式调度，可以在 `goroutine` 执行时间过长的时候强制让出调度，执行其他的 `goroutine`。接下来看看具体怎么实现的。基于 `go1.15 linux amd64`。

先看一个常规的例子

```go
func main() {
    // 只有一个 P 了，目前运行主 goroutine
    runtime.GOMAXPROCS(1)

    // 创建一个 G1 , call $runtime.newproc -> runqput(_p_, newg, true)放入到本地队列，注意这里的 next 参数为 true, 代表放在队列头部
    go f1()

    // 因为只有一个 P 主 goroutine继续运行

    // 创建一个 G2 , 同上面，也是加入到队列头部，这时候本地队列的顺序就是 G2 G1
    go f2()

    // 等待 f1 f2的执行, gopark 主goroutine, GMP调度可运行的 G
    // 按顺序调用 G2 G1
    // 所以不管执行多少次，结果都是
    // This is f2
    // This is f1
    // success
    time.Sleep(100 * time.Millisecond)
    fmt.Println("success")
}
```

如果将 `runtime.GOMAXPROCS(1)` 改成 `runtime.GOMAXPROCS(4)` 即多核CPU，你就会发现 `f1` 和 `f2` 交替执行，没有明确的先后。因为此时有4个 `P`，两个 `M` 可以分别运行自己的 `P` 并行执行 `f1` 和 `f2`。

既然每次都是 `f2` 先执行，那在 `f2` 中加入一个死循环会怎么样呢？

```go
func f1() {
    fmt.Println("This is f1")
}

func f2() {
    // 死循环
    for {

    }
    fmt.Println("This is f2")
}

func main() {
    runtime.GOMAXPROCS(1)

    go f1()
    go f2()

    time.Sleep(100 * time.Millisecond)
    fmt.Println("success")
}

// This is f1
// success
```

这段代码在 go 1.14 中可以正常执行，但 go 1.13 中则会卡住。因为之前版本得执行函数时才会检查调度，对于 `f2` 里这种简单的 `for` 循环则无能为力。



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
- 抢占式调度器 1.2 ~ 今
  - 基于**协作**的抢占式调度器 1.2 ~ 1.13
    - 通过编译器，在编译时向函数头部写入**抢占检查**代码，在函数调用时减产当前 `goroutine` 是否发起了抢占请求
    - `goroutine` 可能会因为垃圾回收和循环长时间占用资源导致程度卡住
  - 基于**信号**的抢占式调度器 1.14 ~ 今
    - 实现基于信号的真·抢占式调度
    - 垃圾回收在扫描栈时会触发抢占调度
    - 抢占的时间点不够多，还不能覆盖全部的边缘情况



# 强制抢占

操作系统使用的是强制抢占，即线程执行超过一定时间后，不管是否执行完，强制换上别的线程，以实现公平





# Go 1.2 协作式调度 / 请求式抢占

- Go 调度在 go1.2 实现了抢占，应该更精确的称为请求式抢占，那是因为 Go调度器的抢占和OS的线程抢占比起来很柔和，不暴力，不会说线程时间片到了，或者更高优先级的任务到了，执行抢占调度。
- Go 的抢占调度柔和到只给 `goroutine` 发送1个抢占请求（把抢占 `flag` 设为 `true` ），`goroutine` 每次执行某个函数前，会先检查这个 `flag`（因此如果 `goroutine` 里没有函数，就不会粗发这个检查，会一直占用CPU，不调度给其他线程）。
- 抢占请求需要满足 2个条件中的1个：
    1. G进行系统调用超过20us
    2. G运行超过10ms。
- 调度器在启动的时候会启动一个单独的线程sysmon，它负责所有的监控工作，其中1项就是抢占，发现满足抢占条件的G时，就发出抢占请求。



### 实现机制

- 编译阶段，编译器会在一些有明显消耗的函数头部插入一些栈增长检测代码，用来做扩栈和调度相关的判断。
- 函数执行时，通过检测 `g.stackguard0==0`，来判断是否要扩栈。该标志也用来判断是否该做调度，如果 `g.stackguard0 == stackPreempt`，则执行调度。
  - 调度时还会检查 `gcwating` 标志，如果 `gcwating==1`, (表示 `GC` 正在等待执行 `STW` 操作)，便会让出当前 `P`
- 因此，对于非函数调用 (例如某个 `G` 执行的是简单 `for` 循环)，即使设置了抢占标志，该 `G` 也会一直霸占 `M` ，无法完成调度





# Go 1.14 信号式抢占

- 程序启动时，会注册绑定 `SIGURG` 信号及处理方法 `runtime.doSigPreempt`
- `sysmon` 会间隔性检测超时的 `P`，然后发送 `SIGURG` 信号
- `M` 收到信号后休眠执行的 `goroutine` 并且进行重新调度
- 详见 `sysmon` 章节



# 调度时机
在四种情形下，`goroutine` 可能会发生调度，但也并不一定会发生，只是说 Go scheduler 有机会进行调度。

1. 使用关键字 `go` 创建一个新的 `goroutine`，Go scheduler 会考虑调度
2. GC。由于进行 GC 的 `goroutine` 也需要在 M 上运行，因此肯定会发生调度。当然，Go scheduler 还会做很多其他的调度，例如调度不涉及堆访问的 `goroutine` 来运行。GC 不管栈上的内存，只会回收堆上的内存
3. 系统调用。当 `goroutine` 进行系统调用时，会阻塞 `M`，所以它会被调度走，同时一个新的 `goroutine` 会被调度上来
4. 内存同步访问。atomic，mutex，channel 操作等会使 `goroutine` 阻塞，因此会被调度走。等条件满足后（例如其他 `goroutine` 解锁了）还会被调度上来继续运行



# 抢占是否影响性能 

抢占分为 `_Prunning` 和 `Psyscall`，`Psyscall` 抢占通常是由于阻塞性系统调用引起的，比如磁盘IO、cgo。`Prunning` 抢占通常是由于一些类似死循环的计算逻辑引起的。

过度的发送信号来中断 `M` 进行抢占多少会影响性能的，主要是软中断和上下文切换。在平常的业务逻辑下，很难发生协程阻塞调度的问题。



# 信号的原理

我们对一个进程发送信号后，内核把信号挂载到目标进程的信号 `pending` 队列上去，然后进行触发软中断设置目标进程为 `running` 状态。当进程被唤醒或者调度后获取CPU后，从内核态转到用户态时检测是否有 `signal` 等待处理，等进程处理完后会把相应的信号从链表中去掉。

在Linux中的posix线程模型中，线程拥有独立的进程号，可以通过 `getpid()` 得到线程的进程号，而线程号保存在 `pthread_t` 的值中。而主线程的进程号就是整个进程的进程号，因此向主进程发送信号只会将信号发送到主线程中去。如果主线程设置了信号屏蔽，则信号会投递到一个可以处理的线程中去。

注册的信号处理函数都是线程共享的，一个信号只对应一个处理函数，且以最后一次为准。子线程也可更改信号处理函数，且随时都可改。



# 参考

[深度解密Go语言之 scheduler](https://www.cnblogs.com/qcrao-2018/p/11442998.html)

[单核CPU下Go语言调度及抢占式调度的实现](https://www.jianshu.com/p/9238bf572b56)

[golang 异步抢占例子](https://zhuanlan.zhihu.com/p/216118842)

[抢占式调度](https://blog.csdn.net/czadev/article/details/109650559)

[峰云就她了-go1.14基于信号的抢占式调度实现原理](http://xiaorui.cc/archives/6535)

[查漏补缺之Goroutine的调度](https://www.cnblogs.com/yudidi/p/12380871.html)

[朴素的心态-单核CPU下Go语言调度及抢占式调度的实现](https://www.jianshu.com/p/9238bf572b56)