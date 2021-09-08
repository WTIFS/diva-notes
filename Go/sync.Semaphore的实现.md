# 信号量

信号量是并发编程中常见的一种同步机制，在需要控制访问资源的线程数量时就会用到信号量。其实信号量就是一种变量或者抽象数据类型，用于控制并发系统中多个进程对公共资源的访问，访问具有原子性。信号量主要分为两类：

- 二进制信号量 `binary semaphore`：其值只有两种 `0 `或者 `1`，相当于互斥量，当值为 `1 `时资源可用，当值为 `0` 时，资源被锁住，进程阻塞无法继续执行。
  - 在 `Linux` 系统中，二进制信号量又称互斥锁 `Mutex`。
- 计数信号量 / 一般信号量 `counting semaphore / general semaphore`：信号量是一个任意的整数，起始时，如果计数器的计数值为`0`，那么创建出来的信号量就是不可获得的状态，如果计数器的计数值大于`0`，那么创建出来的信号量就是可获得的状态，并且总共获取的次数等于计数器的值。



信号量是由操作系统来维护的，信号量只能进行两种操作等待和发送信号，操作总结来说，核心就是 `PV` 操作：

- `P` 原语：`P` 是荷兰语 `Proberen` (测试) 的首字母。为阻塞原语，负责把当前进程由运行状态转换为阻塞状态，直到另外一个进程唤醒它。操作为：申请一个空闲资源 (把信号量减1)，若成功，则退出；若失败，则该进程被阻塞；
- `V` 原语：`V` 是荷兰语 `Verhogen` (增加) 的首字母。为唤醒原语，负责把一个被阻塞的进程唤醒，它有一个参数表，存放着等待被唤醒的进程信息。操作为：释放一个被占用的资源 (把信号量加1)，如果发现有被阻塞的进程，则选择一个唤醒之。

在信号量进行 `PV` 操作时都为原子操作，并且在 `PV` 原语执行期间不允许有中断的发生。

`PV` 原语对信号量的操作可以分为三种情况：

- 把信号量视为某种类型的共享资源的剩余个数，实现对一类共享资源的访问
- 视信号量为一个加锁标志，实现对一个共享变量的访问
- 把信号量用作进程间的同步



运行方式：

1. 初始化信号量，给与它一个非负数的整数值。
2. 运行 `P`，即 `wait`，信号量 `S` 的值将被减少。企图进入临界区的进程，需要先运行 `wait()`。当信号量 `S` 减为负值时，进程会被阻塞住，不能继续；当信号量 `S` 不为负值时，进程可以获准进入临界区。
3. 运行 `V`，即 `signal`，信号量 `S` 的值会被增加。结束离开临界区的进程，将会运行 `signal()`。当信号量 `S` 不为负值时，先前被阻塞住的其他进程，将可获准进入临界区。

我们一般用信号量保护一组资源，比如数据库连接池、一组客户端的连接等等。**每次获取资源时都会将信号量中的计数器减去对应的数值，在释放资源时重新加回来。当遇到信号量资源不够时尝试获取的线程就会进入休眠，等待其他线程释放归还信号量**。如果信号量是只有0和1的二进位信号量，那么，它的 `P/V` 就和互斥锁的 `Lock/Unlock` 一样了。





# Go语言中的信号量表示

`Go` 内部使用信号量来控制 `goroutine` 的阻塞和唤醒，比如互斥锁 `sync.Mutex` 结构体定义的第二个字段就是一个信号量。


```go
type Mutex struct {
    state int32
    sema  uint32 // -> 信号量其实就是个int
}
```





# 使用

`semaphore.Weight`，这个结构的作用比较像限流器，不过它限的是协程数。先看个梨子：

```go
func main() {
    urls := []string{"1", "2", "3", "4"}

    s := semaphore.NewWeighted(3)               // 声明配额为3的信号量
    var w sync.WaitGroup
    for _, u := range urls {
        w.Add(1)
        go func(u string) {
            s.Acquire(context.Background(), 1)  // 占用1个信号量资源
            doSomething(u)
            s.Release(Weight)                   // 解除信号量占用
            w.Done()
        }(u)
    }
    w.Wait()
    
    fmt.Println("All Done")
}
```

这样就可以限制最多3个协程在跑了。





# 实现原理

我们来看一下信号量 `semaphore.Weighted` 的数据结构：

```go
type Weighted struct {
    size    int64         // 最大资源数
    cur     int64         // 当前已被使用的资源
    mu      sync.Mutex    // 互斥锁，对字段的保护
    waiters list.List     // 等待的调用者队列，先进先出
}

type waiter struct {
	n     int64           // 需要占用的配额
	ready chan<- struct{} // 用于唤醒调用者
}
```



#### Acquire 获取信号量

取不到就等待，而且还要检测传递进来的 `context.Context` 对象是否发送了超时过期 / 者取消的信号。

```go
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
    s.mu.Lock() // 加锁保护临界区
   
    if s.size-s.cur >= n && s.waiters.Len() == 0 { // 有剩余配额，则扣减配额后直接返回
        s.cur += n
        s.mu.Unlock()
        return nil
    }
  
    if n > s.size { // 请求的资源数大于能提供的最大的资源数，这个任务处理不了
        s.mu.Unlock()
        <-ctx.Done()
        return ctx.Err()
    }
    
    // 现存资源不够, 需要把调用者加入到等待队列中
    ready := make(chan struct{}) 
    w := waiter{n: n, ready: ready} // 创建了一个 chan, 这样可以通过 close(chan) 的方式对其通知
    elem := s.waiters.PushBack(w)   // 把 w 放到队尾
    s.mu.Unlock()
  
    // 等待
    select {
    case <-ctx.Done(): // context 的 Done 被关闭
        err := ctx.Err()
        s.mu.Lock()
        select {
        case <-ready: // 如果被唤醒了，忽略ctx的状态
            err = nil
            
        default: // 通知其他等待者
            isFront := s.waiters.Front() == elem
            s.waiters.Remove(elem)
            
            if isFront && s.size > s.cur { // 唤醒其它的 waiters, 检查是否有足够的资源
                s.notifyWaiters()
            }
        }
        s.mu.Unlock()
        return err
        
    case <-ready: // 等待者被唤醒了
        return nil
    }
  }
```



#### NotifyWaiters 通知等待者

`notifyWaiters` 方法会逐个检查队列里等待的调用者，如果现存资源够等待者请求的数量n，或者是没有等待者了，就返回。

```go
func (s *Weighted) notifyWaiters() {
    for { // 遍历所有 waiter
        next := s.waiters.Front()
        if next == nil {
            break // 没有等待者了，直接返回
        }
  
        w := next.Value.(waiter)
        if s.size-s.cur < w.n {
            // 如果现有资源不够队列头调用者请求的资源数，比如配额还剩10，但要占用11，就退出。所有等待者会继续等待
            // 这里还是按照先入先出的方式处理是为了避免饥饿
            break
        }

        s.cur += w.n
        s.waiters.Remove(next)
        close(w.ready) // close(chan)，唤醒等待者
    }
}
```

`notifyWaiters` 方法是按照先入先出的方式唤醒调用者。当释放 100 个资源的时候，如果第一个等待者需要 101 个资源，那么，队列中的所有等待者都会继续等待，即使队列后面有的等待者只需要 1 个资源。这样做的目的是避免饥饿，否则的话，资源可能总是被那些请求资源数小的调用者获取，这样一来，请求资源数巨大的调用者，就没有机会获得资源了。

 

#### Release归还信号量资源

`Release `方法会将占用资源放回，并调用 `notifyWaiters` 方法，唤醒等待队列中的调用者。

```go
func (s *Weighted) Release(n int64) {
    s.mu.Lock()
    s.cur -= n        // 释放信号量
    s.notifyWaiters() // 通知等待者
    s.mu.Unlock()
}
```





# 总结

在 `Go` 语言中信号量有时候也会被 `Channel `类型所取代，因为一个 `buffered chan` 也可以代表 n 个资源。不过既然 `Go` 语言通过`golang.orgx/sync` 扩展库对外提供了 `semaphore.Weight` 这一种信号量实现，遇到使用信号量的场景时还是尽量使用官方提供的实现。在使用的过程中我们需要注意以下的几个问题：

- `Acquire` 和  `TryAcquire  `方法都可以用于获取资源，前者会阻塞地获取信号量。后者会非阻塞地获取信号量，如果获取不到就返回 `false`。
- `Release` 归还信号量后，会以先进先出的顺序唤醒队列中的等待者。如果现有资源不够队头的调用者请求的资源数，所有等待者会继续等待。
- 如果一个 `goroutine` 申请较多的资源，由于上面说的归还后唤醒等待者的策略，它可能会等待比较长的时间。





#### 参考

> [KevinYan11 - 信号量的使用方法和其实现原理](https://mp.weixin.qq.com/s/QAMgkj-pDe36leDeGigu4Q)
