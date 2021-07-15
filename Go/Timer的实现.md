# Timer的实现
- `go1.10` 及之前
  - 一个全局四叉堆
    - 最小堆是非常常见的用来管理 timer 的数据结构。在最小堆中，作为排序依据的 key 是 timer 的 `when` 属性，也就是何时触发。即最近一次触发的 timer 将会处于堆顶。
  - 起一个叫 `timeproc` 的协程负责调度 `timer`，不断从堆顶取元素，判断是否到期并触发对应的 `callback`
- `go1.0 ~ go1.13` 
  - 64个全局桶 + 四叉堆
    - 分成了64个四叉堆，降低锁粒度
    - 每个同一个 `timeproc` 协程
- `go1.14` 
  - 不再使用64个桶，四叉堆改放到 `P` 里



## 1. 底层结构

```go
// time/sleep.go
type Timer struct {
    C <-chan Time  // 到时间后会向这个 channel 写数据。是一个有缓冲的 channel, 缓冲区大小为 1。
    r runtimeTimer
}

type runtimeTimer struct {
    pp       uintptr
    when     int64                      // timer 触发的绝对时间，计算方式就是当前时间加上 duration
    period   int64
    f        func(interface{}, uintptr) // timer触发时，需要执行的函数。不可为闭包。
    arg      interface{}                // 传给f的参数
    seq      uintptr
    nextwhen int64
    status   uint32
}
```

`time` 包提供了2种方法来新建 `Timer`：`time.NewTimer` 和 `time.AfterFunc`



### 1.1 NewTimer

```go
// 返回1个Timer，该Timer在 至少 d 时间后向 channel 发送时间
func NewTimer(d Duration) *Timer {
    c := make(chan Time, 1)
    t := &Timer{
        C: c,
        r: runtimeTimer{
            when: when(d),
            f:    sendTime, // 到时间后，向c中写入数据
            arg:  c,
        },
    }
    startTimer(&t.r)
    return t
}
```



`timer` 对象构造好后，接下来就调用了 `startTimer` 函数

```go
// runtime/time.go
// startTimer adds t to the timer heap. -> timer 实际上是用的大根堆，每个P一个这样的堆
// 通过link做方法映射，time/sleep.go里调用的time.startTimer其实是runtime包里的。
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
    ok := cleantimers(pp) && doaddtimer(pp, t) // 清理P中的timer，并将当前timer加入P
    unlock(&pp.timersLock)
    wakeNetPoller(when) // 根据时间就近来唤醒netpoll
}

```



## sleep 的实现



#### 参考

> [cyhone - Golang 定时器底层实现深度剖析](https://www.zhihu.com/people/cyhone/posts)