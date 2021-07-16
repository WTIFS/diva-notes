# Timer的实现
- `go1.10` 及之前
  - 一个全局最小堆
    - 最小堆是非常常见的用来管理 timer 的数据结构。在最小堆中，作为排序依据的 key 是 timer 的 `when` 属性，也就是何时触发。即最近一次触发的 timer 将会处于堆顶。
  - 再起一个叫 `timeproc` 的协程负责调度 `timer`，不断从堆顶取 `timer`，判断是否到期并触发对应的 `callback`
- `go1.0 ~ go1.13` 
  - 64个全局桶 + 四叉堆
    - 分成了64个四叉堆，降低锁粒度
    - 每个桶一个 `timeproc` 协程
- `go1.14` 
  - 不再使用64个桶，四叉堆改放到 `P` 里
  - 不再使用 `timeproc` 协程调度，改为使用 `netpoll`，其工作原理和 `epoll` 一样
- 其它方案
  - 时间轮，比如将 1秒 分成长度为 1000 的环形队列，则每段表示 1ms，每 1ms 扫描一下该段上的 `timer`
  - 对于每秒一次的 `ticker` 类型，只需要设置一个一秒后的 `timer`，这个 `timer` 的回调是一秒后再写入一个 `timer`，就可以实现，类似递归



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



### 1.1 NewTimer

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
    ok := cleantimers(pp) && doaddtimer(pp, t) // 清理 P 中的 timer，并将当前 timer 加入 P 的时间堆里
    unlock(&pp.timersLock)
    wakeNetPoller(when)                        // 根据时间就近来唤醒netpoll
}
```



## sleep 的实现

1. 每个 `goroutine` 底层的 `G` 对象上，都有一个 `timer` 属性，这是个 `runtimeTimer` 对象，专门给 `sleep` 使用。当第一次调用 `sleep` 的时候，会创建这个 `runtimeTimer`，之后 `sleep` 的时候会一直复用这个 `timer` 对象。
2. 调用 `sleep` 时，触发 `timer` 后，直接调用 `gopark`，将当前 `goroutine` 挂起。
3. 调用 `callback` 的时候，不是像 `timer` 和 `ticker` 那样使用 `sendTime` 函数，而是直接调 `goready` 唤醒被挂起的 `goroutine`。





#### 参考

> [cyhone - Golang 定时器底层实现深度剖析](https://www.zhihu.com/people/cyhone/posts)