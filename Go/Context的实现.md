# 深入理解Golang之context

# 1. 常用场景

1. 上下文控制，全链路排查跟踪，如传递trace_id、用户名
2. 树状并发情景中的协程控制（某层任务取消后，所有的下层任务都会被取消，上层及平层不受影响）
3. 超时控制



# 2. 底层结构

```go
// 其实是个接口
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}   // 被取消或者超时时候返回的一个close的channel，子函数需要对其监听。(像全局广播，因为是close的，所有协程都能收到)
    Err() error              // 被取消的原因
    Value(key interface{}) interface{} // 如果当前节点上无法找到key所对应的值，就会向上去父节点里找，直到根节点
}


// A canceler is a context type that can be canceled directly. The
// implementations are *cancelCtx and *timerCtx.
type canceler interface {
	  cancel(removeFromParent bool, err error)
	  Done() <-chan struct{}
}
```

为啥不将这里的 `canceler` 接口 `与Context` 接口合并呢？况且他们定义的方法中都有 `Done` 方法，可以解释得通的说法是，源码作者认为 `cancel` 方法并不是 `Context` 必须的，根据最小接口设计原则，将两者分开。像 `emptyCtx` 和 `valueCtx` 不是可取消的，所以他们只要实现 `Context` 接口即可。



库里头提供了4个`Context`实现，来供大家玩耍

```go
// 完全空的Context，用作树的根节点，其实是个int
type emptyCtx int

// 继承自Context，并实现了canceler接口
type cancelCtx struct {
    Context
    mu    sync.Mutex
    done chan struct{}
    
    children map[canceler]struct{} // 记录可取消的孩子节点
    err error
}

// 继承自cancelCtx，加了个timer，以支持超时处理
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.
    deadline time.Time
}

// 继承自Context，加了个kv
// 携带关键信息，为全链路提供线索，比如接入elk等系统，需要来一个trace_id，valueCtx就非常适合做这个事
type valueCtx struct {
    Context
    key, val interface{}
}
```



# 3. 具体实现

## cancelCtx

### cancel方法的实现

```go
// closedchan is a reusable closed channel.
var closedchan = make(chan struct{})

func init() {
	close(closedchan)
}

// 其实主要就做了一件事：把 c.done 这个 channel 关闭掉，这样所有监听 <- c.Done() 的协程都会收到通知
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	c.mu.Lock() // 加锁
	c.err = err // 设置取消原因
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
	for child := range c.children { // 将子节点依次取消
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()
}
```


### WithCancel
```go
// 新建一个cancel context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c) // 将新建的c加到其父节点的child列表里
	return &c, func() { c.cancel(true, Canceled) }
}

func propagateCancel(parent Context, child canceler) {
    // parentCancelCtx这个函数会从parent开始向上找，直到找到一个可取消的context为止
    // 即找到parent(含)最近的可取消祖先节点
    if p, ok := parentCancelCtx(parent); ok { 
        p.mu.Lock()
        if p.children == nil {
            p.children = make(map[canceler]struct{})
        }
        p.children[child] = struct{}{} // 将child放到这个祖先节点的children列表里，以便祖先取消时能一次调用child的cancel方法
        p.mu.Unlock()
    } else {
        // 如果没找到可用的祖先节点，就新起一个协程，监听父节点的Done事件
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}

// 用作查祖先节点时的key，值为0没用，用的时候都是取址，把地址当常量用
var cancelCtxKey int

// 找到parent(含)最近的可取消祖先节点
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
	return p, true
}

// 根据key查找value，如果找不到则递归从父节点向上找
func (c *cancelCtx) Value(key interface{}) interface{} {
	if key == &cancelCtxKey {
		return c
	}
	return c.Context.Value(key) // 递归向上找
}
```



## timerCtx

### WithTimeout & WithDeadline

`WithDeadline` 内置了定时器，到期就执行 `cancel` 

`Withtimeout` 实际上是 `Withdeadline` 的语法糖，使用 `timeout` 参数算了下 `deadline` 而已

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	propagateCancel(parent, c)
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.err == nil {
        // 这里设了定时器，到期执行c.cancel方法
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```



# 4. net/http中的实际应用

1. 首先`Server`在开启服务时会创建一个`valueCtx`, 存储了`server`的相关信息，之后每建立一条连接就会开启一个协程，并携带此`valueCtx`。

```go
func (srv *Server) Serve(l net.Listener) error {    
    baseCtx := context.Background() 
  
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)    // ServerContextKey = "http-server"
    for {        
        rw, e := l.Accept()        
        c := srv.newConn(rw)
        go c.serve(ctx)  // 开启新协程处理连接，并传递ctx
    }
}
```

2. 建立连接之后会基于传入的`context`创建一个`valueCtx`用于存储本地地址信息，之后在此基础上又创建了一个`cancelCtx`，然后开始从当前连接中读取网络请求，每当读取到一个请求则会将该`cancelCtx`传入，用以传递取消信号。一旦连接断开，即可发送取消信号，取消所有进行中的网络请求。

```go
// Serve a new connection.
func (c *conn) serve(ctx context.Context) {
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr()) // 本地地址
  
    ctx, cancelCtx := context.WithCancel(ctx)
  	c.cancelCtx = cancelCtx
	  defer cancelCtx()
  
    for {
        w, err := c.readRequest(ctx)
        serverHandler{c.server}.ServeHTTP(w, w.req)        
    }
}
```

3. 读取到请求之后，会再次基于传入的`context`创建新的`cancelCtx`,并设置到当前请求对象`req`上，同时生成的`response`对象中`cancelCtx`保存了当前`context`取消的方法。

```go
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {
    req, err := readRequest(c.bufr, keepHostHeader)
    ctx, cancelCtx := context.WithCancel(ctx)
    req.ctx = ctx

    w = &response{
        conn:          c,
        cancelCtx:     cancelCtx,
        req:           req,
        reqBody:       req.Body
    }
    return w, nil
}
```

在整个`server`处理流程中，使用了一条`context`链贯穿`Server`、`Connection`、`Request`，不仅将上游的信息共享给下游任务，同时实现了上游可发送取消信号取消所有下游任务，而下游任务自行取消不会影响上游任务。

另外有两点值得注意：

1. `context` 的取消是柔性的，上游任务仅仅使用 `context` 通知下游不再需要，不会直接干涉和中断下游任务的执行，由下游任务自行处理取消逻辑。
2. `context `是线程安全的，其本身是不可变的（immutable），且并发情况下会加锁。



# 参考
[hudingyu - 深入理解Golang之context](https://juejin.cn/post/6844904070667321357)

[咖啡色的羊驼 - 由浅入深聊聊Golang的context](https://blog.csdn.net/u011957758/article/details/82948750)