# 阻塞 IO

服务端为了处理客户端的连接和请求的数据，写了如下代码。

```go
listenfd = socket();   // 打开一个网络通信端口
bind(listenfd);        // 绑定
listen(listenfd);      // 监听
while(1) {
    connfd = accept(listenfd);  // 阻塞建立连接
    int n = read(connfd, buf);  // 阻塞读数据
     doSomeThing(buf);          // 利用读到的数据做些什么
    close(connfd);              // 关闭连接，循环等待下一个连接
}
```



# 参考

[闪客sun - 你管这破玩意叫 IO 多路复用？]( https://mp.weixin.qq.com/s/YdIdoZ_yusVWza1PU7lWaw)