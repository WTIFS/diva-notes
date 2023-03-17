# 1. nginx 进程模型

nginx 的进程模型采用 master/worker 模型。nginx启动时，主进程作为master 进程，通过配置中 worker 进程数量 N，fork 出 N 个 worker 子进程。

master 进程的作用:

- 管理进程
- 接收信号并转发给所有 worker 子进程。
- 监控worker进程状态 (如 worker 崩溃后，master 会捕获到事件，启动新的 worker)。

worker 进程的作用：

- 接收 master 进程传递信号，并响应（如配置变更后重载等）。
- 处理网络连接，数据读写等。
- woker 进程一般设置为跟 `CPU` 核心数一致。nginx 的 woker 进程在同一时间可以处理的请求数只受内存限制，可以处理多个请求。

程序启动后，主进程 (master进程) 解析配置参数，将配置中需要 listen 的 port 存储到列表中，然后遍历列表，创建 socket、bind 端口、listen socket，完成初始化。然后创建 worker 进程，worket 进程启动后会继承父进程的所有文件描述符，所以 worker 进程拥有了 master 所有 listen 状态的文件描述符列表。

worker 进程启动后，循环检查 `epoll_wait` 处理对应的 handle 事件。



##### 热重启

通过 `nginx -s reload` 命令可以热重启 nginx 进程，其原理为：通过 master/worker 这种模型，master 可以在接收到重载命令后使用新配置新建 worker 子进程，新的连接都有新进程来处理；老 worker 处理完旧连接后关闭，就完成了热重启。



##### 惊群

多个 worker 会将从 master 进程继承的 listen 状态的 fd 加入到 `epoll` 中。如果一个 `fd` 被同时加入到多个 `worker` 进程中，会出现多个 `worker` 被唤醒处理同一个 `fd` 的情况，即惊群现象（内核2.6版本之前），会导致不可预估的错误；同时唤醒多个 worker 处理将导致性能下降。为了防止同一个 `fd` 被同时加入到多个 worker 进程中，nginx 使用了互斥锁，获取到互斥锁的进程将 `fd` 加入到自身进程的 `epoll` 中。



##### 多路复用

每进来一个请求，会有一个 `worker` 进程去处理。`worker` 向 `RS` 转发请求后，就把这个连接放入 `epoll` 红黑树中，然后处理别的连接。

等那个请求返回了，会触发回调，worker 再从就绪队列中取出那个事件返回。





## 为什么 Nginx 不使用多线程？

 首先，作为高性能负载均衡，稳定性非常重要。由于多线程共享同一地址空间，一旦出现内存错误，所有线程都会被内核强行终止，这会降低系统的可用性；

 其次，Nginx的模块化设计允许第三方代码嵌入到核心流程中执行，这虽然大大丰富了Nginx生态，却也引入了风险。

因此，Nginx宁肯选用多进程模式使用多核CPU，而只以多线程作为补充。



#### 参考

[民工哥 - Nginx 如何实现高并发？常见的优化手段有哪些](https://mp.weixin.qq.com/s/aM7n3C8m-YydNBlJhUXprA)

[NGINX开源社区](https://zhuanlan.zhihu.com/p/420043253)