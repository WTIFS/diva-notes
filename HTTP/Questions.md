**在浏览器中输入url后发生了什么**

1. 浏览器查找当前 URL 是否存在缓存
   1. DNS 解析 URL 对应的 IP。
2. 根据IP建立 TCP 连接。
3. 浏览器发起 HTTP 请求。请求经过 IP层、MAC层的封装，组成以太网帧，通过硬件网卡发出去
4. 经过中间设备的转发，最终到达服务器
5. 服务器一层一层地去掉各层封装，向上传输，直到应用层。应用处理请求，并响应
6. 浏览器接收响应。

6. 渲染页面，构建DOM树。



**常见的HTTP Headers**

`Accept`: 浏览器端可以接受的MIME类型。例如：`Accept: text/html `

`Accept-Encoding`: 浏览器申明自己可接收的编码方法，通常指定压缩方法。例如： `Accept-Encoding: gzip, deflate`

`Accept-Language`: 浏览器申明自己接收的语言。例如：`Accept-Language: en-us`

`Accept-Charset`: 浏览器可接受的字符集

`Content-Length`: 消息正文长度

`Content-Type`: 请求体结构。例如：`Content-Type: application/x-www-form-urlencoded`

`Cookie`: 用于位置服务端会话状态。通常由服务端返回，客户端写入本地，请求时再带上。格式类似 kv 结构，多个 kv 以分号分隔

`Connection`: 控制长连接

- `Connection: keep-alive`: HTTP 1.1 默认值，请求完成后，客户端和服务器之间的 TCP连接不会关闭
- `Connection: close`: 请求完成后， TCP连接就会关闭。下次请求需要重建链接

`Host`: HTTP 1.1 唯一的必传字段，用于指定资源的主机和端口号

`Referer`: 告诉服务器我是从哪个链接过来的，服务器可以以此做统计

`User-Agent`: 客户端使用的操作系统和浏览器的名称和版本



