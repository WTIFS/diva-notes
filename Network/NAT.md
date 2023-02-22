我们查看自己的IP地址时，一般都是 192.168.X.X 或是 10.X.X.X。这类地址都是内网地址/私有IP地址/局域网地址。这些地址经过NAT路由器转换为公网IP地址后，才可以和外网连接。这样尽管我们在自己家和别人家可能内网IP都是192.168.1.3，但上层路由器的公网不一样，就可以最大化利用IP地址空间。



## 实现方式

NAT的实现方式这里介绍三种，即静态转换Static NAT、动态转换Dynamic NAT和端口复用 (Port Address Translation，PAT)。

#### 1. 静态NAT

将内部本地地址与内部全局地址进行一对一的明确转换。

这种方法主要用在内部网络中有对外提供服务的服务器，如WEB、MAIL服务器时。该方法的缺点是需要独占宝贵的合法IP地址。

#### 2. 动态NAT

保存一个未使用地址的队列，需要分配时取一个出来，不用时回收放回去。

#### 3. 端口复用

用 IP地址+端口号 的形式进行转换，使多个私网用户可以共用个一个公网IP地址进行访问外网。



## 内网IP地址段

在 RFC1918文档中定义了私有地址的范围。它们不会出现在广域网中，只会出现在局域网内。

A类地址：10.0.0.0 ~ 10.255.255.255（255^3=1600W个）

B类地址：172.16.0.0 ~ 172.31.255.255

C类地址：192.168.0.0 ~ 192.168.255.255（255*255=6W个）

一般一条街或者一个小区用C类地址，公司需要的IP数量会更大，一般就用A类和B类。