#### 在浏览器中输入url后发生了什么

1. 解析 `URL`，获取域名、资源名，生成 `HTTP` 请求
2. `DNS` 解析 `URL` 对应的 `IP`
3. 根据IP建立 `TCP` 连接（四层）
4. 浏览器发起 `HTTP` 请求。请求经过 `TCP`、`IP` 层（三层）、`MAC` 层（二层）的封装，组成以太网帧
5. 通过硬件网卡将包转为电信号，通过网线发出去
6. 经过交换机（二层，基于 `MAC`）、路由器（三层，基于 `IP`）的转发，最终到达服务器
7. 服务器一层一层地去掉各层封装，向上传输，直到应用层。应用处理请求，并响应
8. 浏览器接收响应。
9. 渲染页面。





#### 常见的HTTP Headers

`Accept`: 浏览器端可以接受的 `MIME` 类型。例如：`Accept: text/html `

`Accept-Encoding`: 浏览器申明自己可接收的编码方法，通常指定压缩方法。例如： `Accept-Encoding: gzip, deflate`

`Accept-Language`: 浏览器申明自己接收的语言。例如：`Accept-Language: en-us`

`Accept-Charset`: 浏览器可接受的字符集

`Content-Length`: 消息正文长度

`Content-Type`: 请求体结构。例如：`Content-Type: application/x-www-form-urlencoded`

`Cookie`: 用于位置服务端会话状态。通常由服务端返回，客户端写入本地，请求时再带上。格式类似 kv 结构，多个 kv 以分号分隔

`Connection`: 控制长连接

- `Connection: keep-alive`: HTTP 1.1 默认值，请求完成后，客户端和服务器之间的 TCP连接不会关闭
- `Connection: close`: 请求完成后， TCP连接就会关闭。下次请求需要重建连接

`Host`: HTTP 1.1 唯一的必传字段，用于指定资源的主机和端口号

`Referer`: 告诉服务器我是从哪个链接过来的，服务器可以以此做统计

`User-Agent`: 客户端使用的操作系统和浏览器的名称和版本





#### 设计一个基于 UDP 差不多可靠的算法

`TCP` 和 `UDP` 其实是对成本和质量的不同取舍。`TCP` 重质量但成本大。`UDP` 重成本小但质量差。

`HTTP3` 使用的 `QUIC` 实际上就是基于 `UDP` 的。

1. 可靠连接：增加握手机制
2. 数据完整性：增加 `CRC / Checksum` 校验和
3. 乱序：增加 `SEQ` 序列号
4. 丢包：增加 `ACK` 机制





#### 一些缩写

`MSS`: Max Segment Size，最大报文段长度

`RTT`: Round Trip Time往返时延






#### 参考

[小林coding](https://www.zhihu.com/question/437193010/answer/2065582348)

