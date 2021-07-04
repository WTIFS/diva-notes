## **浏览器的GET和POST**

HTTP 最早被用来做浏览器与服务器之间交互 HTML 和表单的通讯协议。这里特指浏览器中**非**Ajax的HTTP请求。

浏览器用 GET 请求来获取一个html页面/图片/css/js等资源；用 POST 来提交一个 `<form>` 表单，并得到一个结果的网页。



浏览器将 GET 和 POST 定义为：



**GET**

读取一个资源。比如 Get 到一个 html 文件。反复读取不应该对访问的数据有副作用。

因为 GET 是读取，就可以对 GET 请求的数据做缓存 / 书签。

这个缓存可以做到浏览器本身上（彻底避免浏览器发请求），也可以做到代理上（如nginx），或者做到server端（用Etag，至少可以减少带宽消耗）



**POST**

在页面里 `<form> ` 标签会定义一个表单。点击其中的 submit 元素会发出一个 POST 请求让服务器做一件事。这件事往往是有副作用的，不幂等的。





## 接口中的GET和POST

接口中就没有那么多限制了，二者仅仅是 `Method` 不同而已。

你可以给 GET 请求加上 body；也可以让 POST 一半参数放在 queryString 里，另外一半放 body 里；甚至还可以让所有的参数都放 Header 里

当然，太自由也带来了另一种麻烦，前后端可能需要经常讨论接口参数放在哪里传，太不标准了。于是就有了一些列接口规范/风格。其中名气最大的当属 REST。

REST 约定，GET 和 POST 分别对应于资源的 **获取** 和 **创建**。这样仅通过看 HTTP 的 `Method` 就可以明白接口的用途，一目了然。





## 其他

**关于安全性**

我们常听到 GET 不如 POST 安全，因为 POST 用 body 传输数据，而 GET 用 url 传输，如果将密码放在 queryString 里，肉眼就可以明显的看到。

但是从攻击的角度，无论是 GET 还是 POST 都不够安全，因为 HTTP 本身是**明文协议**。每个HTTP请求和返回的每个byte都会在网络上**明文传播**，不管是url，header还是body。

为了避免传输中数据被窃取，必须做从客户端到服务器的端端加密。业界的通行做法就是 **https**，即用 TLS/SSL 协议协商出的密钥，对明文 http 加密。

TLS / SSL 仍然属于应用层，是在 http package 外面的一层，套上 TLS/SSL 后，再包 TCP / IP / 以太网那些层。



**关于编码**

常见的说法有，比如GET的参数只能支持ASCII，而POST能支持任意 binary，包括中文。但其实从上面可以看到，GET 和 POST实际上都能用url和body。因此所谓编码确切地说应该是 http 中 url 用什么编码，body用什么编码。

url 中使用的是 `Percent Encoding`，即一大坨 % 和 16进制组成的序列。

body 相对好些，因为有个 `Content-Type` 来比较明确的定义。比如：

```http
POST xxxxxx HTTP/1.1
...
Content-Type: application/x-www-form-urlencoded ; charset=UTF-8
```

这里 `Content-Type` 会同时定义请求 body 的格式（application/x-www-form-urlencoded）和字符编码（UTF-8）。这样服务器就能知道 body 的编码了，解析不易出错。



**浏览器的POST需要发两个请求吗？**

属于客户端优化的范畴，有的客户端对 POST 请求会分为2次发送，一次发送请求头，如果服务器校验过了，再发送剩下的。

这属于客户端实现的细节，和 GET / POST 本身无关。



**关于URL的长度**

HTTP 协议本身对 URL 长度并没有做任何规定。实际的限制是由客户端/浏览器以及服务器端决定的。

比如我们常说的2048个字符的限制，其实是IE8的限制。

服务端如 apache / nginx 等也可以通过参数限制。

不限制的话，过长的 url + 高并发 会占用大量内存，可能会挤爆服务器。开发使用过长的 url 完全是给自己埋坑。





## 总结

1. HTTP 最早被设计为用来做浏览器与服务器之间做交互的通讯协议。对浏览器来说：
   1. GET 和 POST 的用途不同，GET 用于读取资源；POST 用于提交数据。
   2. 触发方式不同：GET 触发是由用户输入网址 / 点击超链接发起的；POST 是提交表单时发起的。
   3. 数据格式不同：GET 没有 body，携带参数只能通过 url 里的 queryString；POST 也可以通过queryString，但一般是通过 body。
2. 对接口来说，二者仅仅是 `Method` 不同而已。
   1. 你可以用 GET 提交数据，POST 获取数据。也可以给 GET 带上 body
   2. 太自由也会带来问题，因此需要标准来进行约定，比如  RESTFul，约定使用 GET 获取资源，POST 创建资源



#### 参考

[大宽宽 - GET 和 POST 到底有什么区别](https://www.zhihu.com/question/28586791/answer/767316172)
