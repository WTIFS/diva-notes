### Mutex结构体

```go
// mutex 是一种互斥锁
// mutex 的零值是未加锁的互斥锁
// mutex 在第一次使用之后不能进行复制
type Mutex struct {
	state int32  // 状态位 
	sema  uint32 // 信号量，用来控制等待的 goroutine 的阻塞，休眠，唤醒
}

// 上面 state 的可选值
const (
	mutexLocked = 1 << iota     // 持有锁的标记
	mutexWoken                  // 唤醒标记
	mutexStarving               // 饥饿标记
	mutexWaiterShift = iota     // 阻塞等待的数量
	starvationThresholdNs = 1e6 // 饥饿阈值 1ms
)
```



### Lock函数

```go
// 加锁
// 如果锁已经被使用，调用goroutine阻塞，直到锁可用
func (m *Mutex) Lock() {
	// 快速路径：没有竞争直接获取到锁，修改状态位为加锁
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	// 慢路径（以便快速路径可以内联）
	m.lockSlow()
}
```

`Lock` 函数，分为两个部分， 快速路径先通过 `CAS` 尝试直接获取锁，如果能获取到直接返回；否则进入慢路径的方法。快速路径可以内联进上层函数

这个和 `JAVA` 里的 `ReentrantLock` 重入锁的实现类似。



#### 快速路径

快速路径的逻辑很简单：尝试 `CAS` 修改变量，修改成功则视为加锁成功。



#### 慢路径

如果快速路径加锁失败，则进入慢路径，大家排队阻塞等待加锁，每个等待者叫 `waiter`，放入一个队列里等待。



##### 正常模式：非公平锁

正常模式下 `waiter` 是先入先出，从队列中弹出一个 `waiter` 唤醒后，但该 `waiter` 不会直接获取锁，而是新来的请求锁的 `goroutine` 进行竞争。通常新来的 `goroutine` 更易获取锁（新的 `goroutine` 正在`cpu`上运行，而被唤醒的 `waiter` 还要进行调度才能进入状态），所以 `waiter` 大概率抢不过，这个时候 `waiter` 会被放回队列的头部重新等待，如果等待的时间超过了 `1ms`，这个时候 `Mutex` 就会进入饥饿模式。



##### 饥饿模式：公平锁

当 `Mutex` 进入饥饿模式之后，新来的 `goroutine` 会直接放入队列的尾部，不会和老的 `waiter` 竞争，这样很好的解决了老的 `goroutine` 一直抢不到锁的场景。

当队列只剩一个 `goroutine` 并且等待时间没有超过 `1ms`，这个时候 `Mutex` 会重新恢复到正常模式。



这段代码很难懂，大概能看出分正常模式和饥饿模式两种逻辑。

```go
func (m *Mutex) lockSlow() {
	var waitStartTime int64 // 记录请求锁的初始时间
	starving := false       // 饥饿标记
	awoke := false          // 唤醒标记
	iter := 0               // 自旋次数
	old := m.state          // 当前所的状态
	for {
        // 正常模式
        // 尝试自旋获取锁
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {		
		    // 在临界区耗时很短的情况下提高性能
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 && atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
			    awoke = true
			}
			runtime_doSpin()
    		iter++
            old = m.state // 更新锁的状态
     		continue
		}
        
		new := old
		if old&mutexStarving == 0 { // 非饥饿状态进行加锁
			new |= mutexLocked
		}
        
		if old&(mutexLocked|mutexStarving) != 0 { // 等待者数量+1
			new += 1 << mutexWaiterShift
		}
		
		// 加锁的情况下切换为饥饿模式
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
        // goroutine 唤醒的时候进行重置标志
		if awoke {
			new &^= mutexWoken
		}
                
        // 设置新的状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			if old&(mutexLocked|mutexStarving) == 0 {
				break 
			}
            // 判断是不是第一次加入队列
			// 如果之前就在队列里面等待了，加入到队头
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
            // 阻塞等待
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
            // 检查锁是否处于饥饿状态
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
            // 如果锁处于饥饿状态，直接抢到锁
			if old&mutexStarving != 0 {
				// 设置标志，进行加锁并且waiter-1
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				// 如果是最后一个的话清除饥饿标志
                if !starving || old>>mutexWaiterShift == 1 { // 退出饥饿模式
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}
}
```







#### 参考

> [crabman - golang-浅析mutex](https://zhuanlan.zhihu.com/p/340536378)
