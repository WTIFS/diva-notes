## 几个术语和缩写

```text
cip：Client IP，客户端地址
vip：Virtual IP，LVS实例IP
rip：Real IP，后端RS地址
RS: Real Server 后端真正提供服务的机器
LB： Load Balance 负载均衡器
LVS： Linux Virtual Server
sip： source ip
dip： destination ip
```





## LVS的几种转发模式

- DR模型 -- （Director Routing-直接路由）
- NAT模型 -- (NetWork Address Translation-网络地址转换)
- fullNAT -- （full NAT）
- ENAT --（enhence NAT 或者叫三角模式/DNAT，阿里云提供）
- IP TUN模型 -- (IP Tunneling - IP隧道)





## DR模型(Director Routing--直接路由)

1. 请求流量 (sip 200.200.200.2, dip 200.200.200.1) 先到达 LVS
2. LVS 根据负载策略选择一个 RS，然后将这个网络包的 MAC 地址修改成该RS的 MAC
3. 然后丢给交换机，交换机将这个包给选中的 RS
4. 选中的 RS 看到MAC地址是自己的、dip 也是自己的，愉快地手下并处理、回复
5. 回复包（sip 200.200.200.1， dip 200.200.200.2）
6. 经过交换机直接回复给 client 了（不再走LVS）

##### 优点

- DR模式是性能最好的一种模式，入站请求走LVS，回复报文绕过LVS直接发给Client

##### 缺点

- 要求LVS和rs在同一个vlan
- RS需要配置vip同时特殊处理arp
- 不支持端口映射

##### 为什么要求LVS和RS在同一个vlan（或者说同一个二层网络里）

因为DR模式依赖多个RS和LVS共用同一个VIP，然后依据MAC地址来在LVS和多个RS之间路由，所以LVS和RS必须在一个vlan或者说同一个二层网络里





## NAT模型 (NetWork Address Translation - 网络地址转换)

1. client 发出请求（sip 200.200.200.2，dip 200.200.200.1）
2. 请求包到达lvs，lvs修改请求包为（dip rip）
3. 请求包到达 RS， RS 回复（sip rip，dip 200.200.200.2）
4. 这个回复包不能直接给client，因为rip不是 VIP 会被reset掉
5. 但是因为lvs是网关，所以这个回复包先走到网关，网关有机会修改sip
6. 网关修改sip为VIP，修改后的回复包（sip 200.200.200.1，dip 200.200.200.2）发给client

##### 优点

- 配置简单
- 支持端口映射（看名字就知道）
- RIP一般是私有地址，主要用户LVS和RS之间通信

##### 缺点

- LVS和所有RS必须在同一个vlan
- 进出流量都要走LVS转发
- LVS容易成为瓶颈
- 一般而言需要将 VIP 配置成 RS 的网关





#### 参考

[阿里云云栖号 - 就是要你懂负载均衡--lvs和转发模式](https://zhuanlan.zhihu.com/p/72150660)