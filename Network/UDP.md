#### TCP协议和UDP协议之间的分别有哪些有缺点？在工作中如何选择？

TCP的优点：

- 可靠：通过三次握手、四次挥手，提供可靠的连接，能够保证数据的传输完整性和有序性
- 流控&拥塞：支持流控制和拥塞控制，能够适应网络负载和拥塞情况
- 全双工：支持全双工通信，能够实现双向数据传输

TCP的缺点：

- 相比于UDP，TCP的开销更大，需要建立连接、维护状态等操作，占用更多的网络带宽和系统资源
- TCP的延迟较高，建立连接、关闭连接等操作都需要一定的时间
- 不能很好地适应高延迟或者高丢包的网络环境



UDP是一种无连接的协议，不需要在发送数据之前建立连接。UDP协议通过数据包的源IP地址和端口号和目的IP地址和端口号来进行通信，每个数据包都是独立的，没有状态和顺序之类的要求。因此，UDP协议不需要进行连接的建立和断开，也就没有三次握手的概念。

UDP的优点和缺点和TCP反过来就是了

在选择TCP或UDP协议时，需要根据具体的应用场景和要求来决定。如果要求数据传输的完整性和有序性，或者需要实现可靠的双向通信，就应该选择TCP协议；如果要求实时性和低延迟，或者能够容忍数据的丢失和乱序，像音视频流传输这种大量单向数据传递的场景，就可以选择UDP协议。





#### 如何改造UDP协议使之可以保证可靠的数据？

UDP协议是一种无连接、不可靠的协议，它不提供数据的可靠传输保证。如果想对UDP协议进行改造，以实现可靠的数据传输，可以考虑以下几种方法：

##### 1. 应用层增加ACK确认机制、重传机制

应用层增加ACK确认机制；

应用层增加重传机制：发送方发送数据时，设置一个定时器。如果在定时器到期之前没有收到确认消息，则认为数据包丢失，需要重新发送。接收方接收到重复的数据包时，可以直接忽略或发送ACK确认消息。

##### 2. 数据校验和机制

在数据包中添加校验和，用于检验数据在传输过程中是否发生损坏或篡改。接收方在接收到数据包时，可以对校验和进行验证，如果检验失败，则认为数据包已损坏或被篡改，需要请求重传。

##### 3. 窗口控制机制

可以在协议中添加窗口控制机制，用于控制发送方和接收方的数据流量。通过设置窗口大小、滑动窗口等参数，可以实现对数据流量的控制和调整。同时，还可以通过窗口控制机制实现对丢失数据包的重传和超时机制的调整。

##### 4. 前向纠错机制

可以在数据包中添加冗余信息，用于实现前向纠错。发送方可以在数据包中添加冗余信息，接收方在接收到数据包时，可以通过冗余信息对数据进行纠错。这种机制可以提高数据的可靠性和完整性，减少重传的次数。