调度器在启动的时候会启动一个单独的线程 `sysmon`，它负责所有的监控工作，其中一项就是抢占，发现满足抢占条件的 `G` 时，就发出抢占请求。 

```go
func sysmon() {
    //无限循环
    for {
        //渐进式休眠，最短20us，最长10ms
		if idle == 0 { 
			delay = 20
		} else if idle > 50 { //前50轮休眠时间都是20us，50轮后double
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		usleep(delay) //休眠
        
        //retake对所有的G进行重整
        if retake(now) != 0 {
            idle = 0 //retake成功则重置轮数
        } else {
            idle++ //否则轮数+1
        }
    }
}
```



# 发送调度信号

```go
//retake对所有的G进行重整
func retake(now int64) uint32 {
    //加锁
    lock(&allpLock)

    for i := 0; i < len(allp); i++ {
        _p_ := allp[i]
        pd := &_p_.sysmontick
        s := _p_.status
        
        if s == _Prunning || s == _Psyscall {
            t := int64(_p_.schedtick)
            
            // G对应的schedtick和schedwhen跟监控的不一致时，需要重新更新一些monitor的数值
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t) //调度时效
                pd.schedwhen = now //上次的检测时间
            } else if pd.schedwhen+forcePreemptNS <= now {
                // forcePreemptNS = 10ms，如果协程独占P超过10ms，就需要进行抢占了
                preemptone(_p_)
                sysretake = true
            }
        }
    ...
    }
}

//抢占
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	gp := mp.curg
    
    // 标记G的抢占标识为true
    // (在1.14之前的版本协作式抢占中，这个标记设置好了，就只能干等着，直到 morestask (函数调用)或者gopark（IO操作，例如channel之类）才会检查)
	gp.preempt = true

	// 1.14及之后，会直接调用preemptM信号通知线程，所谓信号式抢占
	if preemptMSupported && debug.asyncpreemptoff == 0 {
		_p_.preempt = true
		preemptM(mp)
	}
}

//发出抢占信号
func preemptM(mp *m) {
    // CAS保证并发安全
    if atomic.Cas(&mp.signalPending, 0, 1) {
        //发出信号
        //const sigPreempt = _SIGURG，SIGURG是Linux下几乎没用的一个信号量，被go拿来当作抢占信号了
		signalM(mp, sigPreempt)
	}
}

```



# 接收调度信号

```go
//程序初始化时，监听SIGURG信号
//src/runtime/proc.go
func mstartm0() {
    initsig(false)
}

//src/runtime/signal_unix.go
func initsig(preinit bool) {
        // 对于一个需要设置 sighandler 的信号，会通过 setsig 来设置信号对应的动作（action）：
        setsig(i, funcPC(sighandler))
}

//runtime/preempt.go
//这个函数里包含了对各种信号量的处理，其中就包括SIGURG
func sighandler(sig uint32, info *siginfo, ctxt unsafe.Pointer, gp *g) {
    if sig == sigPreempt && debug.asyncpreemptoff == 0 {
        doSigPreempt(gp, c)
    }
}

func doSigPreempt(gp *g, ctxt *sigctxt) {
    // 判断异步安全点
    if wantAsyncPreempt(gp) {
        if ok, newpc := isAsyncSafePoint(gp, ctxt.sigpc(), ctxt.sigsp(), ctxt.siglr()); ok {
            // Adjust the PC and inject a call to asyncPreempt.
            ctxt.pushCall(funcPC(asyncPreempt), newpc)
        }
    }

    // Acknowledge the preemption.
    atomic.Xadd(&gp.m.preemptGen, 1)
    atomic.Store(&gp.m.signalPending, 0)
}

// asyncPreempt saves all user registers and calls asyncPreempt2.
// When stack scanning encounters an asyncPreempt frame, it scans that
// frame and its parent frame conservatively.
// 这个函数是用汇编实现的，实际调用了asyncPreempt2
func asyncPreempt()

//go:nosplit
func asyncPreempt2() {
    gp := getg()
    gp.asyncSafePoint = true
    if gp.preemptStop {
        mcall(preemptPark) //里面调用的runtime.schedule()，见MPG章节里的调度
    } else {
        mcall(gopreempt_m) //里面调用的goschedImpl，goschedImpl调用的runtime.schedule()
    }
    gp.asyncSafePoint = false
}
```



# 收到信号后的处理

`preemptPark` 方法会解绑 `M` 和 `G` 的关系，封存当前协程，执行 `runtime.schedule()` 来重新调度，获取可执行的协程，至于被抢占的协程后面会去重启。

```go
// runtime/proc.go
// preemptPark parks gp and puts it in _Gpreempted.
func preemptPark(gp *g) {
	casGToPreemptScan(gp, _Grunning, _Gscan|_Gpreempted)     //修改G的状态为抢占中
	dropg()                                                  //解绑M和G
	casfrom_Gscanstatus(gp, _Gscan|_Gpreempted, _Gpreempted) //修改G的状态为被抢占
	schedule()                                               //调度
}

//解绑M和G
func dropg() {
	_g_ := getg()

	setMNoWB(&_g_.m.curg.m, nil) //将G的M设置为nil
	setGNoWB(&_g_.m.curg, nil)   //将M的curg设置为nil
}
```

`goschedImpl` 操作就简单的多，把当前协程的状态从 `_Grunning` 正在执行改成 `_Grunnable` 可执行，使用`globrunqput` 方法把抢占的协程放到全局队列里，根据 `MPG` 的协程调度设计，全局队列要后于本地队列被调度。
```go
func gopreempt_m(gp *g) {
	goschedImpl(gp)
}

func goschedImpl(gp *g) {
	casgstatus(gp, _Grunning, _Grunnable) //修改G的状态为可运行
	dropg()                               //解绑M和G
	lock(&sched.lock)                     //加锁
	globrunqput(gp)                       //把g放入全局队列
	unlock(&sched.lock)                   //解锁
	schedule()                            //调度
}
```



# 参考

[峰云就她了 - go1.14基于信号的抢占式调度实现原理](http://xiaorui.cc/archives/6535)