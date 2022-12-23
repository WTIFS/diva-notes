Go 语言里的 TCP 连接是怎样建立的呢？

要分客户端和服务端两部分看



<img src="./assets/listen.jpeg" alt="img" style="zoom:50%;" />





# 客户端

客户端一般通过如下代码发送请求：

```go
func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:3000")
    conn.Write([]byte(data)) // 向服务端发送数据
    n,err := conn.Read(buf)  //读取服务端端数据
}
```



`net.Dial()` 底层实际调用的是系统函数 `socket()` 和 `connet()`。这里不详细解析，主要看服务端





# 服务端

服务端一般通过如下代码接收客户端请求：

```go
func main()  {
    listen, err := net.Listen("tcp",":8080") // 创建监听 socket
    for {
        conn, errs := listen.Accept()       // 接收客户端连接
        go handle(conn) 					// 一个 goroutine 处理一个连接
    }
}
```


## net.Listen() 的实现
`net.Listen()` 经过层层调用，最底层实际调用的是如下系统函数：
1. `socket()` 创建 socket
2. `bind()` 绑定 socket 与监听地址
3. `listen()` 监听 socket
4. `epollcreate()` 创建 epoll 对象
5. `epollctl()` 将监听 socket fd 加入到 epoll 红黑树里进行监听

```go
// net/dial.go
func Listen(network, address string) (Listener, error) {
    var lc ListenConfig
    return lc.Listen(context.Background(), network, address)
}

// lc.Listen()
func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
    sl := &sysListener{}
    case *TCPAddr:
        l, err = sl.listenTCP(ctx, la)
    case *UnixAddr:
        l, err = sl.listenUnix(ctx, la)
    }
}
```



```go
// net/tcpsock_posix
// sysListener.listenTCP()
func (sl *sysListener) listenTCP(ctx context.Context, laddr *TCPAddr) (*TCPListener, error) {
    fd, err := internetSocket(ctx, sl.network, laddr, nil, syscall.SOCK_STREAM, 0, "listen", sl.ListenConfig.Control)
}
```



```go
// net/ipsock_posix.go
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
    return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr, ctrlFn)
}
```



```go
// net/sock_posix.go
// socket()
// laddr: local address, radd: remote address
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
    s, err := sysSocket(family, sotype, proto) // 调用 syscall.Socket() 创建系统 socket 对象
    fd, err = newFD(s, family, sotype, net)    // new 一个 fd 结构体
    if laddr != nil && raddr == nil {          // 远端地址为空，表示监听。服务端进入这个 if 分支
        switch sotype {
            case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
                fd.listenStream(laddr, listenerBacklog(), ctrlFn) // 绑定端口并监听
        }
    }
}

// listenStream()
func (fd *netFD) listenStream(laddr sockaddr, backlog int, ctrlFn func(string, string, syscall.RawConn) error) error {
    syscall.Bind(fd.pfd.Sysfd, lsa)    // syscall.Bind() 将监听地址绑定到 socket 上
    listenFunc(fd.pfd.Sysfd, backlog)  // syscall.Listen() 监听，backlog 参数控制连接队列长度，取自系统参数 /proc/sys/net/core/somaxconn
    fd.init()                          // epoll 对象初始化和跟踪
}
```

`fd.init()` 里主要是对 epoll 对象的创建和跟踪，实现如下：
```go
// net/fd_unix.go
// fd.init()
func (fd *netFD) init() error {
    return fd.pfd.Init(fd.net, true)
}

func (fd *FD) Init(net string, pollable bool) error {
    err := fd.pd.init(fd)
}
```

```go
// poll/fd_poll_runtime.go
func (pd *pollDesc) init(fd *FD) error {
    serverInit.Do(runtime_pollServerInit) 
    ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
}
```

`runtime_pollServerInit()` 函数通过 `go:linkname` 注释由 `runtime/netpoll.go/poll_runtime_pollServerInit()` 实现，底层调用 `epoll_create1()` 函数，创建 epoll 对象
```go
// runtime/netpoll.go
//go:linkname poll_runtime_pollServerInit internal/poll.runtime_pollServerInit
func poll_runtime_pollServerInit() {
    netpollGenericInit()
}

func netpollGenericInit() {
    netpollinit() // 根据操作系统有不同的实现，linux 下是 epoll，MacOS 下是 kqueue
}
```

`netpollinit()` 函数在 linux 系统下的实现文件为 `runtime/netpoll_epoll.go`
```go
// runtime/netpoll_epoll.go
func netpollinit() {
    epfd = epollcreate1(_EPOLL_CLOEXEC) // 创建 epoll fd；使用汇编实现，实际调用 linux epoll_create1() 函数
}
```

`runtime_pollOpen()` 函数由 `runtime/netpoll.go/net_runtime_pollOpen()` 实现，底层调用 `epoll_ctl()` 函数，将 socket fd 放入 epoll 对象中监听，以便在和客户端的连接建立时得到通知
```go
// runtime/netpoll.go
//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
    errno := netpollopen(fd, pd)
}
```

```go
// runtime/netpoll_epoll.go
func netpollopen(fd uintptr, pd *pollDesc) int32 {
    return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev) // 实际调用 linux epoll_ctl()
}
```

## listen.Accept() 的实现
`listen.Accept()` 的逻辑主要是：
1. 调用系统函数 `accept()` 等待并接收连接
2. 连接到来后，调用 `epollcreate()` 和 `epollctl()` （和上面一样）管理 连接socket

```go
// net/tcpsock.go
func (l *TCPListener) Accept() (Conn, error) {
    c, err := l.accept() // accept() 函数会阻塞式的等待下一个连接
}

func (ln *TCPListener) accept() (*TCPConn, error) {
    fd, err := ln.fd.accept()
}
```

```go
// net/fd_unix.go
func (fd *netFD) accept() (netfd *netFD, err error) {
    d, rsa, errcall, err := fd.pfd.Accept() // 等待连接

    netfd, err = newFD(d, fd.family, fd.sotype, fd.net) // 为连接新建 fd
    netfd.init() // 这个和前面的 init() 一样，创建和维护 epoll 对象
}

func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	
}
```
