[TOC]

# grpc

grpc 是自带名字解析、负载均衡和连接池策略的。使用 gprc.Dial 方法实际上建立的是到远端服务的虚拟连接，它实际是在服务的每台实例上建立了一个连接（sub connection），默认使用轮询策略从多个实例中进行负载均衡。



# 魔改流派：mrpc/trpc/dubbo-go/tars

这个流派主要是对 protoc 的插件进行修改，使之生成的 pb 文件发生改变，在其中注入自己的逻辑

无论服务端还是客户端都需要先通过 `Init` 方法初始化，从配置中心拉取配置



**服务端**

`NewServer` 方法返回一个 `Server` 对象，提供 `Serve、Stop、SetHandler`（HTTP注册路由处理函数）、`RegisterService` 给 gRPC 使用的注册处理函数方法

`protoc` 会在 pb 文件里生成 `RegisterXXXServer` 方法。在 `NewServer` 方法之后需要通过这个 `RegisterXXXServer` 方法将 `handler` 函数注册进 pb

```go
s := rpc.NewServer()
pb.RegisterXXXServer(s, 业务类)
```

`tars` 的写法更简洁，自带一个全局默认 server 对象，初始化并将业务类注册进这个对象后，直接调用包级的 `Run()` 方法启动服务



**客户端**

魔改了 `protoc-gen` 的代码，pb 文件里生成 `NewXXXClientProxy` 代码，返回一个代理，使用该代理调用具体的远端方法，在代理内部执行节点发现、流量调度、拦截器等功能

```go
var cli = pb.NewHiClientProxy(opts) // 魔改后生成的 pb 代码
//-------------------------------------------------------
cli.Hi() // 里面实际调用 cli.pool.GetConn().Invoke() 方法，即 grpc 的 Invoke 方法
```

`dubbo-go` 的实现更为复杂，一方面魔改了 `protoc-gen-go`，在 pb 中额外生成了 struct 类型的结构体，字段为 RPC 函数签名；另一方面又通过配置文件去进行客户端函数与远端函数的关联，里面用了反射去做方法的替换



# 原生流派：go-zero/tt

**服务端**

`rest.NewServer` 方法返回一个 HTTP Server 对象，该对象提供 3 个方法：Start、Stop、AddRoutes

`zrpc.NewServer` 方法返回一个 RPC Server，提供 2 个方法：Start、Stop。Start(registerFunc) 方法用注册函数作为参数，在 Start 内部做了方法的注册

还定义了服务组 `ServiceGroup`，结构为 []Service，Service 为包含 Start 和 Stop 两个方法的接口



**客户端**

使用了 gRPC。`zrpc.NewClient` 方法返回一个 `gRPC client` 对象，该 client 会和远端建立一个连接。使用 client.Conn 作为参数新建 pb 里具体方法的 client 对象后调用对应方法：

```
client := NewClient()
cli := pb.NewHiClient(client.Conn())
cli.Hi()
```



# 自研流派：go-micro/trpc

`NewService` 方法返回 `Service` 接口，接口提供 `Init()、Run()、Client()、Server()` 方法

`Init()` 里初始化了鉴权、消息队列、注册中心、客户端、服务端、存储、采样等

`Run()` 里调用了 `Start()` 方法，并监听信号，在监听到结束信号后调用 Stop()方法



**服务端**

`Service` 里的 `Server()` 方法返回的即为服务端，也是个接口，提供 `Start()、Stop()、Handle()` 方法



**客户端**

没有使用 gRPC，自己实现的传输。`Service` 里的 Client()方法返回的即为客户端，也是个接口，提供 `Call()` 方法用于远程调用。业务方在 `request` 里设置好远端服务名和方法名后，使用该方法调用。方法内部会执行节点选择、编码发送等