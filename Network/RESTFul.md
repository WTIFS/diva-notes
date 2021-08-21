REST，（Representation State Transfer）表示层状态转化
简单来说，就是用URI表示资源，用HTTP方法（GET，POST，PUT，DELETE）对资源进行操作

# 特点
资源、统一接口、URI和无状态



## 1. 资源

网络上的一个实体，可以是图片、文字等。JSON是现在最常用的资源表示格式。



## 2. 统一接口

数据的元操作，即CURD操作，分别对应于HTTP方法：
GET、POST、PUT、DELETE



## 3. URI

**URI** Universal Resource Identifier 统一资源标识符
**URL** Universal Resource Locator 统一资源定位符
URL是URI的子集，是URI表现方式的一种
每个资源都可以用一个URI表示、访问到



## 4. 无状态

client发送的多次请求之间相互独立，服务器不保存客户端的状态。（如session，session会导致服务无法做分布式，同一客户端发来的请求只能由建立了session的机器处理）。如果确实要保持状态，也应由客户端保存状态（如Cookie）。
比如用户登录后，服务端可以发送一个auth，从第二次请求开始，客户端的每个请求都带上这个auth信息，这种就是无状态请求。