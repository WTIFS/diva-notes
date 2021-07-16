# Timer的实现
- `go1.10` 及之前
  - 一个全局最小堆
    - 最小堆是非常常见的用来管理 timer 的数据结构。在最小堆中，作为排序依据的 key 是 timer 的 `when` 属性，也就是何时触发。即最近一次触发的 timer 将会处于堆顶。
  - 再起一个叫 `timeproc` 的协程负责调度 `timer`，持续检查堆顶 `timer`是否到期，并触发对应的回调
- `go1.0 ~ go1.13` 
  - 上述方案存在锁竞争；且堆体积较大，插入/删除效率低
  - 64个全局桶 + 四叉堆
    - 将所有定时器分布到 64 个最小堆中, 减小每个堆的数据量，插入时用 `P` 的 `id` 取模找堆，降低锁粒度
    - 每个桶一个 `timeproc` 协程
- `go1.14` 
  - 不再使用64个桶，四叉堆改放到 `P` 里
  - 不再使用 `timeproc` 协程调度，改为使用 `netpoll` 来让定时器的到期后直接得到通知，其工作原理和 `epoll` 一样
- 其它方案
  - 时间轮
    - 比如将 1秒 分成长度为 1000 的环形队列，则每段槽表示 1ms，每 1ms 扫描一下该段上的 `timer`
    - 可以像时分秒的设计一样，使用多层时间轮组合，即可表示更多的时间
    - `kafka` 用的就是这种方案
  - 对于持续性定时的 `ticker` 类型，只需要将其从堆/队列里取出，调整下次到期时间，再放回去即可，类似递归



> 以下内容基于 go1.14



## 1. 底层结构

`timer` 主要关注3个属性：用于通知外层时间的 `channel`、用于判断触发时间的 `when`，和回调用的函数 `f`

```go
// time/sleep.go
type Timer struct {
    C <-chan Time  // 到时间后会向这个 channel 写数据。是一个有缓冲的 channel, 缓冲区大小为 1。
    r runtimeTimer
}

type runtimeTimer struct {
    pp       uintptr                    // p的位置
    when     int64                      // timer 触发的绝对时间，计算方式就是当前时间加上 duration
    period   int64                      // 周期时间，适合ticker
    f        func(interface{}, uintptr) // 回调方法。timer触发时，需要执行的函数。不可为闭包。
    arg      interface{}                // 传给f的参数
    seq      uintptr                    // 序号
    nextwhen int64                      // 下次的到期时间
    status   uint32                     // 状态
}

// runtime2.go
type p struct {
    timersLock mutex // ?? 还以为放到P里可以无锁，怎么这里还是有锁
    timers []*timer  // 四叉堆，存放定时器任务
}
```

`time` 包提供了2种方法来新建 `Timer`：`time.NewTimer` 和 `time.AfterFunc`



### 1.1 初始化 NewTimer

`go` 里 `timer` 的回调就是 `sendTime`，即向 `channel` 中发送当前时间。

外层监听这个 `channel`，就可以知道定时器是否到时间了。

```go
// 返回1个Timer，该Timer在 至少 d 时间后向 channel 发送时间
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        r: runtimeTimer{
            when: when(d),
            f:    sendTime, // 回调就是向channel发送一个时间。到时间后，向c中写入数据
            arg:  c,
        },
    }
    startTimer(&t.r)
    return t
}

func sendTime(c interface{}, seq uintptr) {
    select {
    case c.(chan Time) <- Now():
    default:
    }
}
```



`timer` 对象构造好后，接下来就调用了 `startTimer` 函数，将 `timer` 放到当前 `P` 的时间堆里

```go
// runtime/time.go
// startTimer adds t to the timer heap. -> timer 实际上是用的四叉堆，每个 P 里一个这样的堆
// 通过link做方法映射，time/sleep.go 里调用的 time.startTimer 其实是 runtime 包里的。
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
    addtimer(t)
}

func addtimer(t *timer) {
    t.status = timerWaiting
    addInitializedTimer(t)
}

// addInitializedTimer adds an initialized timer to the current P.
func addInitializedTimer(t *timer) {
    when := t.when

    pp := getg().m.p.ptr() 
    lock(&pp.timersLock)
    
    ok := cleantimers(pp) && doaddtimer(pp, t) // 清理最小堆里的失效节点 && 把新建的 timer 放到堆里
    unlock(&pp.timersLock)
    
    wakeNetPoller(when)                        // 唤醒 netpoller 中休眠的线程
}
```



##### **runtime.cleantimers**

对 `P` 中 `timer` 最小堆的头节点进行清理工作

```go
func cleantimers(pp *p) {
	for { // 使用了一个死循环来获取堆顶节点
		
		t := pp.timers[0] // 获取堆顶 timer
      
		switch s := atomic.Load(&t.status); s {   // 判断堆顶节点的状态
		
            case timerDeleted:  // 删除状态
				dodeltimer0(pp) // 删除
		
			case timerModifiedEarlier, timerModifiedLater: // timer 被修改到了更早或更晚的时间
				t.when = t.nextwhen                        // 设置新的 when 字段
				dodeltimer0(pp)                            // 删除
                doaddtimer(pp, t)                          // 设置新的 when 字段后重新放回堆
		}
	}
}
```



##### **runtime.doaddtimer**

`doaddtimer` 函数实际上很简单，主要是将 `timer` 与 `P` 设置关联关系，并将 `timer` 加到最小堆里。

```go
func doaddtimer(pp *p, t *timer) { 
	// Timers 依赖于 netpoller
	// 所以如果 netpoller 没有启动，需要启动一下
	if netpollInited == 0 {
		netpollGenericInit()
	}

	t.pp.set(pp) // 设置 timer 与 P 的关联
	i := len(pp.timers)
    
	
	pp.timers = append(pp.timers, t) // 将 timer 加入到堆中
	siftupTimer(pp.timers, i)        // 调整 timer 在堆中的位置
}
```



##### **runtime.wakeNetPoller**

`wakeNetPoller` 主要是将 `timer` 下次调度的时间和 `netpoller` 的下一次轮询时间相比，如果小于的话，调用 `netpollBreak` 向 `netpollBreakWr` 管道里面写入数据，立即中断 `netpoll`

```go
var (
    epfd int32 = -1                        // epoll descriptor
    netpollBreakRd, netpollBreakWr uintptr // 用来给netpoll中断
)

// 初始化全局的epfd及break的两个读写管道
func netpollinit() {
    epfd = epollcreate1(_EPOLL_CLOEXEC)

    r, w, errno := nonblockingPipe()                // r为管道的读端，w为写端

    errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)  // epollctl 进行监听

    netpollBreakRd = uintptr(r)
    netpollBreakWr = uintptr(w)
}

// 上文添加定时器时，如果定时器的 when 小于 pollUntil 时间，则唤醒正在 netpoll 休眠的线程
func wakeNetPoller(when int64) {
    if atomic.Load64(&sched.lastpoll) == 0 {
        pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
        if pollerPollUntil == 0 || pollerPollUntil > when {
            netpollBreak()
        }
    }
}

// netpollBreakWr 是一个管道，用 write 给 netpollBreakWr 写数据，这样 netpoll 自然就可被唤醒。
func netpollBreak() {
    for {
        var b byte
        n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
    }
}
```



## sleep 的实现

1. 每个 `goroutine` 底层的 `G` 对象上，都有一个 `timer` 属性，这是个 `runtimeTimer` 对象，专门给 `sleep` 使用。当第一次调用 `sleep` 的时候，会创建这个 `runtimeTimer`，之后 `sleep` 的时候会一直复用这个 `timer` 对象。
2. 调用 `sleep` 时，触发 `timer` 后，直接调用 `gopark`，将当前 `goroutine` 挂起。
3. 调用 `callback` 的时候，不是像 `timer` 和 `ticker` 那样使用 `sendTime` 函数，而是直接调 `goready` 唤醒被挂起的 `goroutine`。





#### 参考

> [cyhone - Golang 定时器底层实现深度剖析](https://www.zhihu.com/people/cyhone/posts)
>
> [luozhiyun - Go中定时器实现原理及源码解析](https://www.cnblogs.com/luozhiyun/p/14494540.html)
>
> [rfyiamcool - go1.14基于netpoll优化timer定时器实现原理](http://xiaorui.cc/archives/6483)

