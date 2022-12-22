<img src="./assets/listen.jpeg" alt="img" style="zoom:50%;" />



Go 的服务端接收请求的代码逻辑是：

```go
func main()  {
    listen, err := net.Listen("tcp",":8080") // 创建监听
    for {
        conn, errs := listen.Accept()  // 接收客户端连接
        go handle(conn) 							 // 一个 goroutine 处理一个连接
    }
}
```



`listen.Accept()` 函数层层向下调用：

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
    listenFunc(fd.pfd.Sysfd, backlog)  // syscall.Listen() 监听，backlog 参数控制连接队列长度
}
```

