`profiling` 工具基本分两种思路：

- 插桩
  - 具体实现方式就是在每个函数的开始和结束时计时，然后统计每个函数的执行时间。
  - 插桩计时本身也会带来性能消耗，并且对于正在运行的程序没办法插桩。

- 采样
  - 具体实现是注入正在运行的进程，高频地去探测函数调用栈。根据大数定律探测次数越多的函数运行时间越长。
  - `perf`、`pprof` 工具都是这个思路



# CPU profiling

#### 开启采样

```go
// runtime/pprof/pprof.go
// 开启CPU采样，并设置频率
func StartCPUProfile(w io.Writer) error {
    cpu.profiling = true
	runtime.SetCPUProfileRate(hz)
    go profileWriter(w)
}
```



#### 收集采样

程序在启动时会注册对 `SIGPROF` 信号的监听，收到该信号后就会开始采样。采样逻辑是收集栈调用信息，从中分析出当前正在执行的函数及其调用链，并记录到 `cpuprof.log` 的 `buffer` 缓存中

**疑问：**只看到处理 `SIGPROF` 的地方，没有看到发送的地方

```go
// runtime/proc.go
// 处理 SIGPROF 信号
func sigprof(pc, sp, lr uintptr, gp *g, mp *m) {
    // traceback，收集栈调用信息到 stk 切片里
    // 从栈里可以解析出当前执行的函数、函数调用链等
    n = gentraceback(pc, sp, lr, gp, 0, &stk[0], len(stk), nil, nil, _TraceTrap|_TraceJumpStack)
    
	if prof.hz != 0 {
		cpuprof.add(gp, stk[:n])
	}
}

// 记录采样到 buffer
func (p *cpuProfile) add(gp *g, stk []uintptr) {
    cpuprof.log.write(tagPtr, nanotime(), hdr[:], stk)
}
```



#### 将采样结果写入文件

将采样结果从 `buffer` 写入文件

```go
// runtime/pprof/pprof.go
// 循环调用 readProfile()，读取采样数据，并写入到指定的 cpu.prof 文件中
func profileWriter(w io.Writer) {
	for { // 死循环
		time.Sleep(100 * time.Millisecond) // 每100ms写一次，这个是写死的
		data, tags, eof := readProfile()
		if e := b.addCPUData(data, tags); e != nil && err == nil {
			err = e
		}
	}
}
```





# Mem Profiling

其原理是在内存分配的时候进行采样，将分配的内存及其调用堆栈信息拷贝到一个 `mbuckets` 数组，然后拿这个数组做分析。

```go
// runtime/malloc.go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if rate := MemProfileRate; rate > 0 {
        mp := acquirem()
		profilealloc(mp, x, size) // 这里采样
		releasem(mp)
	}
}
```







#### 参考
[phantom_111 - golang pprof 的原理分析](https://blog.csdn.net/phantom_111/article/details/112547713)

[曹春晖 - pprof 的原理与实现](https://mp.weixin.qq.com/s/1VoZ9dZYk7-yWS3mP0Nc-w)
