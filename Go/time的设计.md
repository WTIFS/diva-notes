### 墙上时钟 和 单调时钟

墙上时钟即计算机系统的时钟，一般设成和NTP（Network Time Protocal，网络时间协议）同步，也可以人为修改，因此程序里如果依赖这个时间的话可能会出现时间倒退的现象。

单调时钟一般在程序开始时设为0、或在计算机启动后设为0。其绝对值没有意义，主要用于时间的比较。



### go 里的 Time

`go` 里使用了三个字段表示一个时间，`wall`，`ext`，和时区 `loc`

```go
type Time struct {
	wall uint64
	ext  int64
	loc *Location
}
```



其中 `wall` 的最高位是个标记位，用于存储是否包含单调时钟的信息。1表示包含，0表示不包含。根据这个位的01值，`wall` 和 `ext` 存储的数字会有不同的含义。

```go
hasMonotonic = 1 << 63 // as a mask
```



第一种情况：`hasMonotonic = 1`

![time 存储格式-1](assets/6b36bd86371d4fbb9ebf0fae0e701113~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

这种情况下 `wall` 里存储1885年以后的墙上时间，`ext` 里存储单调时间。



第二种情况：`hasMonotonic = 0`

![time 存储格式-2](assets/ae5a99bf24bd4403befa946c0f902ca4~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp)

这时由于不存储单调时间了，空间比较富裕，可以存储公元 1年开始的时间。`ext` 用于存储秒级的墙上时间，`wall` 里存储纳秒



像通过 `time.Parse` 得到的时间，就没有单调时钟。`time.Now()` 就有

我们打印 `Time` 结构体时，根据是否有单调时钟，打印出的时间也会有不同的样式：

```go
fmt.Println(time.Now())
// 2022-07-01 15:14:39.576193 +0800 CST m=+0.000061626
// 后面这个 m 就是单调时钟，表示离程序启动的时间

t, _ := time.Parse("2006-01-02", "2022-07-01")
fmt.Println(t)
// 2022-07-01 00:00:00 +0000 UTC
// 这个就没有单调时钟
```





### 为什么这么设计？

网上搜到的都是关于设计的解读，没有搜到关于为什么的解读。

个人认为主要是出于空间占用考虑。通过这种设计，使用 2字节可存储2个时间（墙上和单调时间）；在不需要单调时间时，还可将时间范围拓展至公元 1年。

总体思路就是，使用最小的空间，最灵活的逻辑，存储尽可能多的数据。





### time.Since 的逻辑优化

在计算当前时间 `now()` 和上一个时间 `t` 的时间差时，可以用 `time.Since(t)`，也可以用 `time.Now().sub(t)`

`go1.12` 之后 `time.Since` 对于有单调时钟的 `time` 对象有快速处理：直接使用 `runtime` 层的 `nano time` 计算，可减少一次系统调用，提升性能45%。Benchmark 见 https://github.com/golang/go/commit/fc3f8d43f1b7da3ee3fb9a5181f2a86841620273

```go
// It is shorthand for time.Now().Sub(t).
func Since(t Time) Duration {
	var now Time
	if t.wall&hasMonotonic != 0 {
    // 快速路径：如果 t 包含单调时钟，则使用 runtime 层的 nano_time 来初始化 now，直接使用两个时间的 ext 字段进行相减
    // startNano 是程序启动时即初始化为 time.Now() 的时间
		now = Time{hasMonotonic,              // wall 值直接设为标记位，其余的位用不上
               runtimeNano() - startNano, // ext 使用 runtime 层的 nano_time，后续用这个字段相减来算差值
               nil}
	} else {
		now = Now()
	}
	return now.Sub(t)
}
```



#### 参考

[章鱼编程 - Golang Time 包源码分析1-Time & 时区类实现](https://juejin.cn/post/6890889801600335886)
