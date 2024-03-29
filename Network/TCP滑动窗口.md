# TCP滑动窗口

`TCP` 头里有一个字段叫 `Window`，又叫 `AdvertisedWindow`，这个字段是接收端告诉发送端自己还有多少缓冲区可以接收数据。于是发送端就可以根据这个接收端的处理能力来发送数据，而不会导致接收端处理不过来。

<img src="assets/sliding_window.jpg" alt="img" style="zoom:80%;" />



右图接收端 `LastByteRead` 指向了 `TCP` 缓冲区中读到的位置，`NextByteExpected` 是收到的连续包的最后一个位置，`LastByteRcved` 是收到的包的最后一个位置，我们可以看到中间有些数据还没有到达，所以有数据空白区。

左图发送端的 `LastByteAcked` 指向了被接收端 `ACK` 过的位置（表示成功发送确认），`LastByteSent` 表示发出去了，但还没有收到成功确认的 `ACK`，`LastByteWritten` 指向的是上层应用正在写的地方。

接收端在给发送端回 `ACK` 中会汇报自己的 `Window = 缓冲区大小 – LastByteRcvd – 1`

发送方会根据这个窗口来控制发送数据的大小，以保证接收方可以处理。



下面我们来看一下发送方的滑动窗口示意图：

![img](assets/tcpswwindows.png)

上图中分成了四个部分，分别是：

1. 已收到 `ACK` 确认的数据，可以抛弃了
2. 发了还没收到 `ACK` 的。（2和3黑框圈起的部分是滑动窗口）
3. 在窗口中还没有发出的（接收方还有空间）。
4. 窗口以外的数据（接收方没空间）



下面是个滑动后的示意图。发送端方收到了 `36` 的 `ACK`，前移 `5` 字节，发送 `46-51` 的字节

![img](assets/tcpswslide.png)



---

上面的要记；下面的瞟一眼：

下面我们来看一个接受端控制发送端的图示：

![img](assets/tcpswflow.png)



## Zero Window

上图，我们可以看到一个处理缓慢的接收端是怎么把发送端的 `window` 给降成 `0  `的。如果 `window` 变成 `0` 了，发送端就不发数据了，可以想像成 "Window Closed"。如果发送端不发数据了，接收方一会儿 `Window size` 可用了，怎么通知发送端呢？

为了解决这个问题，`TCP` 使用了 `Zero Window Probe` 技术，缩写为 `ZWP`，称为**窗口探测**，检测可用的 `Window`。

一般这个值会设置成 `3` 次，如果 `3` 次过后还是 `0` 的话，有的 `TCP` 实现就会发 `RST` 把连接断了。如果不设的话，遭遇零窗口攻击时，会无限探测，空耗资源。



## Silly Window Syndrome
`Silly Window Syndrome` 翻译成中文就是 "糊涂窗口综合症"。如上面看到的一样，如果我们的接收方太忙了，来不及取走 `Receive Windows` 里的数据，就会导致发送方越来越小。到最后，如果接收方腾出很少的几个字节并告诉发送方现在的 `window`，发送方便会义无反顾地发送这几个字节。

我们的 `TCP + IP` 头有 `40` 个字节，为了几个字节的数据，搭上 `40` 字节的头，太不经济了。

`Silly Windows Syndrome` 这个现象就像是你本来可以坐 `200` 人的飞机里只坐了一两个人。

要解决这个问题也不难，就是避免对小的 `window size` 做出响应，直到有足够大的 `window size` 再响应，这个思路可以同时实现在发送和接收两端。



#### 接收端

如果收到的数据导致 `window size` 小于某个值，可以直接回 `ACK window=0` ，这样就把 `window ` 给关闭了，阻止了发送方再发数据过来

等到接收端处理了一些数据后 `windows size` 大于等于了`MSS (Maximum Segment Size) 最大报文长度`，或者接收缓冲区有一半为空，就可以把 `window` 打开让发送方发送数据过来。



#### 发送端

如果这个问题是由发送端引起的，那么就会使用著名的 `Nagle` 算法。这个算法的思路也是延时处理，他有两个主要的条件：

1. 要等到 `Window Size >= MSS` 或是 `Data Size >= MSS`
2. 收到之前发送数据的 `ACK` 回包，他才会发数据，否则会攒数据等待发送。

`Nagle` 算法默认是打开的，所以对于一些需要小包场景的程序（如 `telnet` 或 `ssh` 这样的交互性比较强的程序），你需要关闭这个算法





#### 参考

[陈皓 - TCP 的那些事儿（下）](https://coolshell.cn/articles/11609.html)