#### Gosched

用户可以通过在代码里调用 `runtime.Gosched()` 来触发调度。该函数可将当前 `G` 任务暂停，放回全局队列等待调度，让出当前 `M` 去执行其他任务。

```go
// runtime/proc.go
func Gosched() {
  	mcall(gosched_m)
}

// mcall 主要做三件事：
// 1. 保存当前 G 状态，将 SP、PC 寄存器等保存到 G.sched 区域
// 2. 切换到 g0
// 3. 执行 fn
func mcall(fn func(*g))

func gosched_m(gp *g) {
    goschedImpl(gp)
}

func goschedImpl(gp *g) {
    casgstatus(gp, _Grunning, _Grunnable) // 修改 G 的状态为 Grunnable
    dropg()                               // 解绑 M 和 G
    lock(&sched.lock)
    globrunqput(gp)                       // 将 G 放入全局队列
    unlock(&sched.lock)

    schedule()                            // 调度
}
```



#### gopark

与 `Gosched()` 最大的区别在于，少了将 `G` 放回全局队列这步。必须主动调用 `goready()` 恢复运行，否则该任务会遗失。

`goready` 会直接将 `G` 放回 `P.runnext`，使之以最高优先级恢复运行。

```go
// runtime/proc.go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
    mcall(park_m)
}

func park_m(gp *g) {
    _g_ := getg()

    casgstatus(gp, _Grunning, _Gwaiting) 
    dropg()     // 解绑 M 和 G

    schedule()
}
```



#### goready

当通过 `gopark()` 休眠的协程满足条件、可以唤醒后（比如从 `channel` 中收到了数据），运行时会调用 `goready()` 唤醒协程

```go
// runtime/proc.go
func goready(gp *g, traceskip int) {
    systemstack(func() {
        ready(gp, traceskip, true)
    })
}

func ready(gp *g, traceskip int, next bool) {
    _g_ := getg()

    casgstatus(gp, _Gwaiting, _Grunnable)
    runqput(_g_.m.p.ptr(), gp, next)
    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
        wakep()
    }
}

func wakep() {
    startm(nil, true) // 找一个 M 执行 P 的协程
}
```





#### stopTheWorld

在 `GC` 时，用户逻辑必须暂停在一个安全点上，否则会引发很多意外问题。STW 通过通知机制，让 `G` 主动停止。比如，设置 `gcwaiting=1` ，再执行调度函数 `schedule()` 休眠 `M`；向所有 `P` 里正在运行的 `G` 发出抢占调度信号，使其暂停。

```go
// runtime/proc1.go
func stopTheWorld(reason string) {
      semacquire(&worldsema)     // CAS加锁，其它执行STW的函数会阻塞住
      gp := getg()
      systemstack(func() {
            casgstatus(gp, _Grunning, _Gwaiting)  // 设置当前 G 状态为 waiting
            stopTheWorldWithSema()                // 实际STW函数
            casgstatus(gp, _Gwaiting, _Grunning)  // 恢复 G 状态
    })
}

func stopTheWorldWithSema() {
    _g_ := getg()

    lock(&sched.lock)
    sched.stopwait = gomaxprocs        // 需要暂停的 P 的计数
    atomic.Store(&sched.gcwaiting, 1)  // 设置停止标志，schedule 之类的函数会检查这个标志，主动休眠 M
    preemptall()                       // 向所有正在运行的 G 发出抢占调度
    
    _g_.m.p.ptr().status = _Pgcstop    // 暂停当前 P
    sched.stopwait--
    
    // 暂停所有 syscall 状态的 P
    for _, p := range allp {                                      // 遍历全局 P 数组
        s := p.status
        if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) { // 其实就是改了下 P 的状态为 gcstop 
            p.syscalltick++
            sched.stopwait--
        }
    }

    // 暂停空闲 P
    for {
        p := pidleget()
        if p == nil {
            break
        }
        p.status = _Pgcstop // 也是更新状态
        sched.stopwait--
    }
    wait := sched.stopwait > 0
    unlock(&sched.lock)

    // 如果还有 P 没暂停，这里会一直休眠 - 重试调度
    if wait {
        for {
            // wait for 100us, then try to re-preempt in case of any races
            if notetsleep(&sched.stopnote, 100*1000) {
                noteclear(&sched.stopnote)
                break
            }
            preemptall()
        }
    }
}
```





#### 参考

> 雨痕《Go语言学习笔记》
