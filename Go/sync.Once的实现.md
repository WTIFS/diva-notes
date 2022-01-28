`sync.Once ` 的源码还是很少的，首先我们看一下他的结构：

```go
type Once struct {
    done uint32 // 用来标识代码块是否执行过
    m    Mutex  // 互斥锁
}
```



```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 { // 如果未执行过，进入slow
        o.doSlow(f)
    }
    // 如果执行过，直接返回
}

func (o *Once) doSlow(f func()) {
    o.m.Lock() // 加互斥锁
    defer o.m.Unlock()
    if o.done == 0 { // 双重检查。因为可能有多个协程进入 doSlow 函数。第一个协程执行后，后面的协程按顺序还能获取锁，此时再判断一次 done
        defer atomic.StoreUint32(&o.done, 1) // defer 修改 done变量
        f()                                  // 执行函数
    }
}
```

