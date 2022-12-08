# time.Timer的实现

定时任务是我们工作中经常遇到的一个场景。`go` 里可以通过 `time.Timer` / `time.Ticker` 轻松构建定时器用于任务定时执行。不知大家有没有好奇过定时器是如何实现的？ 



# 各版本实现演进

- `go1.10` 及之前
  - 一个全局最小堆
    - 最小堆是非常常见的用来管理 `timer` 的数据结构。在最小堆中，作为排序依据的 `key` 是 `timer` 的 `when` 属性，也就是何时触发。即最近的 `timer` 将会处于堆顶。
  - 再起一个叫 `timeproc` 的协程负责调度 `timer`，通过死循环持续检查堆顶的 `timer` 是否到期，并触发对应的回调
    - `timeproc` 执行时需要阻塞（`for` 死循环）占用 `M`，会挤走 `M` 上本来运行的 `G` 和 `P`，切换代价较高
- `go1.0 ~ go1.13` 
  - 上述方案存在锁竞争；且堆体积较大，插入 / 删除效率低
  - 将全局最小堆改为 `64` 个全局桶 + 最小堆（四叉堆）
    - 将所有定时器打散到 `64` 个最小堆中，减小每个堆的数据量，插入时用 `P` 的 `id` 取模找堆，降低锁粒度
    - 每个桶一个 `timeproc` 协程，仍然存在上下文切换代价高的问题
- `go1.14` 
  - 不再使用 `64` 个桶，四叉堆改放到 `P` 里
    - 但其实之前 `64` 个桶一般也远大于 `P` 的数量，因此这方面的提升效果不大
  - 不再使用 `timeproc` 协程调度，改为在调度时触发定时器的检查，避免了 `timeproc` 的上下文切换、调度
  - 像 `G` 任务一样，当 `P` 本地没有 `timer` 时，可以尝试从其他的 `P` 偷取一些 `timer`
- 其它方案
  - 时间轮：`kafka` 使用
    - 比如将 `1` 秒 分成长度为 `1000` 的环形队列，则每个格子表示 `1ms`
    - 再起个死循环进程，判断当前时间所属的格子上，是否有定时任务需要执行
    - 可以像时分秒的设计一样，使用多层时间轮组合，即可组合表示更多的时间
    - 槽多，可以很大程度减少锁竞争
  - 红黑树：`nginx` 使用
      - 最小堆是把最近的 `timer` 放在堆顶，红黑树则是把最近的 `timer`  放入树最左侧
- 对于持续性定时的 `ticker` 类型，只需要将其从堆/队列里取出，调整下次到期时间，再放回去即可，类似永久性递归



> 以下内容基于 go1.14

## 1. 底层结构

`timer` 主要关注 3 个属性：用于通知外层时间的 `chan`、用于判断触发时间的 `when`，和回调用的函数 `f`

```go
// time/sleep.go
type Timer struct {
    C <-chan Time  // 到时间后会向这个 channel 写数据。是一个有缓冲的 channel, 缓冲区大小为 1。
    r runtimeTimer
}

type runtimeTimer struct {
    pp       uintptr                    // 所属 P
    when     int64                      // timer 触发的绝对时间，计算方式就是当前时间加上 duration
    period   int64                      // 周期时间，适合 ticker
    f        func(interface{}, uintptr) // 回调函数。timer 触发时，需要执行的函数。不可为闭包。
    arg      interface{}                // 传给 f 的参数
    seq      uintptr                    // 序号
    nextwhen int64                      // 下次的到期时间
    status   uint32                     // 状态
}

// runtime2.go
type p struct {
    timersLock mutex // 仍然有锁，因为任务窃取时，timers 也会被别的 P 窃取
    timers []*timer  // 最小四叉堆，存放定时器任务
}
```

`time` 包提供了2种方法来新建 `Timer`：`time.NewTimer` 和 `time.AfterFunc`



### 1.1 初始化 NewTimer

`NewTimer` 里 `timer` 的回调就是 `sendTime`，即向 `channel` 中发送当前时间。

外层监听这个 `channel`，就可以知道定时器是否到时间了。

```go
// time/sleep.go
// 返回1个 Timer，该 Timer 在 至少 d 时间后向 channel 发送时间
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

// 向 channel 里写入时间
func sendTime(c interface{}, seq uintptr) {
    select {
        case c.(chan Time) <- Now():
        default:
    }
}

// 这个和 NewTimer 差不多，只是回调不一样。NewTimer 里回调是写死的，而这里回调是参数，可以自定
func AfterFunc(d Duration, f func()) *Timer {
}
```



`timer` 对象构造好后，接下来就调用了 `startTimer` 函数，将 `timer` 放到当前 `P` 的时间堆里

```go
// runtime/time.go
// startTimer adds t to the timer heap. 这个函数会将 timer 放到 P 的最小堆里
//go:linkname startTimer time.startTimer 通过link做方法映射，time/sleep.go 里调用的 time.startTimer 指向 runtime 包
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
    
    ok := cleantimers(pp) && doaddtimer(pp, t) // 清理堆里的失效节点 & 把新建的 timer 放到堆里
    unlock(&pp.timersLock)
}
```



##### **runtime.cleantimers**

对 `P` 中 `timer` 堆的头节点进行清理工作

```go
func cleantimers(pp *p) {
	for { // 使用了一个死循环来获取堆顶节点
		
		t := pp.timers[0] // 获取堆顶 timer
      
		switch s := atomic.Load(&t.status); s {   // 判断堆顶节点的状态
		
            case timerDeleted:  // 删除状态
				dodeltimer0(pp) // 删除堆顶节点
		
			case timerModifiedEarlier, timerModifiedLater: // timer 被修改到了更早或更晚的时间
				t.when = t.nextwhen                        // 设置新的 when 字段
				dodeltimer0(pp)                            // 删除堆顶节点
        		doaddtimer(pp, t)                          // 设置新的 when 字段后重新放回堆
		}
	}
}
```



##### **runtime.doaddtimer**

`doaddtimer` 函数实际上很简单，主要是将 `timer` 与 `P` 设置关联关系，并将 `timer` 加到堆里。

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





### timer 的运行

`timer` 的运行是交给 `runtime.runtimer` 函数执行的，这个函数会不断检查 `P` 上最小堆堆顶 `timer` 的状态，根据状态做不同的处理。

运行时会根据 `period` 判断该 `timer` 是否为 `ticker` 类型，是否需要反复执行：是的话需要重设下次执行时间，并调整该 ` timer` 在堆中的位置；一次性 `timer` 的话则会删除该 `timer`。最后运行 `timer` 中的回调函数

```go
// runtime/time.go
func runtimer(pp *p, now int64) int64 {
    for {
        t := pp.timers[0] // 不断获取最小堆的堆顶元素
      
        // 判断 timer 状态
        switch s := atomic.Load(&t.status); s {
       
            case timerWaiting:                    
                if t.when > now { return t.when }          // 还没到时间，返回下次执行时间，退出循环
                runOneTimer(pp, t, now)                    // 如果到时间了，则运行该 timer
                return 0                                   
            
            case timerDeleted:                             // 过期 timer，删除           
                dodeltimer0(pp)
          
            case timerModifiedEarlier, timerModifiedLater: // 被修改，需要调整位置
                t.when = t.nextwhen
                dodeltimer0(pp)                            // 删除最小堆的第一个 timer
                doaddtimer(pp, t)                          // 将该 timer 重新添加到最小堆
        }
    }
}

// 运行一次 timer 回调函数
func runOneTimer(pp *p, t *timer, now int64) {
   
    // 表示该 timer 为 ticker，需要再次触发
    if t.period > 0 {  
        
        delta := t.when - now 
        t.when += t.period * (1 + -delta/t.period) // 调整触发时间
        siftdownTimer(pp.timers, 0)                // 调整堆位置
        updateTimer0When(pp)
      
    // 一次性 timer
    } else {
        dodeltimer0(pp) // 删除该 timer
    }  
    
    f := t.f                // 回调函数
    arg := t.arg            // 回调函数的参数
    seq := t.seq
    unlock(&pp.timersLock)  // 这里有点费解，一般是先 lock 再 unlock。这里应该是为了运行较慢的回调时不阻塞其他的 timer，使其他 timer 可以被偷
    f(arg, seq)             // 运行该函数
    lock(&pp.timersLock)
}
```





### timer 的触发 / 唤醒

`timer` 的触发有两种：

- 从调度循环中触发
  - 调用 `runtime.schedule` 执行调度时
  - 调用 `runtime.findrunnable` 获取可执行 `G` / 执行抢占时
- `sysmon` 每轮监控中会触发



**runtime.schedule**

```go
// runtime.schedule
func schedule() {
    _g_ := getg()
  
top:
    pp := _g_.m.p.ptr()
    
    checkTimers(pp, 0) // 检查是否有可执行 timer 并执行，checkTimers 里面会调用前文写到的 runtimer
  
    var gp *g
    if gp == nil {
        gp, inheritTime = findrunnable() // blocks until work is available
    }
    execute(gp, inheritTime)
}
```



**runtime.sysmon**

```go
func sysmon() {
    for {
        now := nanotime()
        next, _ := timeSleepUntil() // 下次需要调度的 timer 到期时间
        if next < now {             // 如果有 timer 到期 
            startm(nil, false)      // 启动新的 M 处理 timer
        }
    }
}
```



另外， `P` 没有 `timer` 了的时候，也会尝试偷窃其余 `P` 的 `timer` 执行。



## sleep 的实现

1. 每个 `goroutine` 底层的 `G` 对象上，都有一个 `timer` 属性，这是个 `runtimeTimer` 对象，专门给 `sleep` 使用。当第一次调用 `sleep` 的时候，会创建这个 `runtimeTimer`，之后 `sleep` 的时候会一直复用这个 `timer` 对象。
2. 调用 `sleep` 时，触发 `timer` 后，直接调用 `gopark`，将当前 `goroutine` 挂起。
3. 它的 `callback` 就是 `goready` ，回调时直接唤醒被挂起的 `goroutine`。



#### 参考

> [cyhone - Golang 定时器底层实现深度剖析](https://zhuanlan.zhihu.com/p/149518569)
>
> [luozhiyun - Go中定时器实现原理及源码解析](https://www.cnblogs.com/luozhiyun/p/14494540.html)
>
> [rfyiamcool - go1.14基于netpoll优化timer定时器实现原理](http://xiaorui.cc/archives/6483)
>
> [unique_id - 关于 Go1.14，你一定想知道的性能提升与新特性](https://gocn.vip/topics/9611)