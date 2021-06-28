# MVCC

`MVCC` 全称 Multi Version  Concurrency Control 多版本并发控制。



# 基本原理

一条记录会有多个版本，每次修改记录都会存储这条记录被修改之前的版本，多版本之间串联起来就形成了一条版本链。

这样不同时刻启动的事务可以无锁地获得不同版本的数据 (普通读)。此时读 (普通读) 写操作不会阻塞，写操作可以继续写，无非就是多加了一个版本，历史版本记录可供已经启动的事务读取。

![Image](assets/640-20210625184601400.png)



#  参考

[是Yes呀 - 关于MySQL的酸与MVCC和面试官小战三十回合](https://mp.weixin.qq.com/s?__biz=MzkxNTE3NjQ3MA==&mid=2247490249&idx=1&sn=4348983da767ff28982324acc1760ce5&chksm=c16277b0f615fea65e1aff38ba209095957a65953018f293ca9a1106e2d56521e09fc121247d&scene=132#wechat_redirect)

