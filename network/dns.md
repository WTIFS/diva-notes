# DNS解析

用户查找域名对应的 IP。

在域名中，**越靠右**的位置表示其层级**越高**。

根域是在最顶层，它的下一层就是 `com` 顶级域，再下面是 `server.com` 所以域名的层级关系类似一个树状结构：

* 根 DNS 服务器
* 顶级域 DNS 服务器 `com`
* 权威 DNS 服务器 `server.com`

![img](../Network/assets/v2-1352f5eee6c13130aaa4204def00ae80\_720w.jpg)

## 域名解析的工作流程

1. 客户端先查缓存，如果没有，则向本地 DNS服务器 发起 DNS请求
2. 本地DNS服务器 询问 根DNS服务器，根DNS服务器 告知 顶级域DNS地址
3. 本地DNS服务器 询问 顶级域DNS服务器，顶级域DNS服务器 告知 权威DNS服务器地址
4. 本地DNS服务器 询问 权威DNS服务器，权威DNS服务器查询IP
5. 本地DNS服务器返回结果给客户端

![preview](../Network/assets/v2-0a69a9e03a6e31d3aaa1af588055d9e0\_r.jpg)

## DNS缓存

浏览器有自己的 DNS 缓存，其中Chrome的过期时间是1分钟，在这个期限内不会重新请求DNS

OS缓存会参考DNS服务器响应的TTL值，但是不完全等于TTL值

各 DNS 服务器上保存的 DNS也是有过期时间的，会刷新
