#### TCP连接四元组

源IP、源端口、目的IP、目的端口



#### 客户端最多能维持多少连接？

客户端有 `2^16 = 65536` 个端口，`1000` 以下的端口是系统保留端口，剩下的端口都可以用来维持连接



#### 服务端最多能维持多少连接？

以一个服务器连接多个客户端为例，客户端 IP 有 `2^32` 个，服务器端口有 `2^16` 个，因此连接数上限是 `2^32 × 2^16 = 2^48`，约两百多万亿。如果客户端也使用 `2^16` 个端口，那连接数上限是 `2 ^ 64`。

但实践中，每维持一条TCP连接，都需要创建一个文件对象。Linux出于安全考虑，在多个配置里都限制了可打开的最大文件数。

假设对最大文件数进行了修改，改为不限文件数。每条TCP链接都需要file、socket等内核对象，一条空连接占3.3KB左右。以 4GB 内存的服务器为例，最多能支撑 4GB / 3.3KB，约100万左右。

>  此时如果另一端发送数据，可能就会导致服务器卡死。



#### 空TCP连接占用内存

空 `TCP` 连接是不占用 `socket` 缓冲区的。只有有数据的活跃连接才占。空连接主要占内存的地方在于 `TCP` 连接本身的结构体，记录了 `fd` 文件、文件夹、`socket` 结构体等。占用大概 `3KB` 左右的内存。

<img src="assets/v2-2651a980c2ff60ea0260260744eeed83_1440w.png" alt="img" style="zoom:67%;" />





#### 参考

[陈硕 - Linux 中每个 TCP 连接最少占用多少内存](https://zhuanlan.zhihu.com/p/25241630)