# TCP Connection
- [TCP Connection](#tcp-connection)
  - [介绍](#介绍)
  - [TCP 状态转换](#tcp-状态转换)
  - [TCP 各种标志位](#tcp-各种标志位)
  - [TCP 三次握手四次挥手](#tcp-三次握手四次挥手)
    - [三次握手](#三次握手)
    - [四次挥手](#四次挥手)

## 介绍

TCP 连接有多种状态，常见状态有:

- LISTENING
  - 监听来自远方的 TCP 端口的连接请求
- CLOSE_WAIT
  - 等待从本地来的连接中断请求
  - 被动关闭方收到FIN报文后，发送ACK确认，但本地应用程序还没有关闭连接，等待应用程序向操作系统发送关闭请求
- TIME_WAIT
  - 等待足够的时间以确保远程 TCP 接收到连接中断请求的确认
  - 主动关闭方在发送最后一个ACK后，进入此状态，等待足够的时间（通常是2MSL），以确保对方收到了最后的ACK，并防止在重用相同端口时出现旧数据包的干扰。
- ESTABLISHED
  - 代表一个打开的连接

> MSL: Maximum Segment Lifetime
> - TCP的MSL（Maximum Segment Lifetime，最大报文生存时间）是指一个TCP报文段在网络上能够存在的最长时间。
> - 它是一个时间参数，超过这个时间，网络中的TCP报文段就会被丢弃。在TCP连接关闭时，处于TIME_WAIT状态的一端会等待2MSL的时间，以确保所有可能在网络中滞留的旧报文段都能消失，从而避免对新建立的连接造成干扰。
>
> MTU: Maximum Transmission Unit
> - 最大传输单元，是指一种通信协议的某一层上面所能通过的最大数据包大小（以字节为单位）。最大传输单元这个参数通常与通信接口有关（网络接口卡、串口等）。例如：Ethernet所能接收最大数据帧是1518字节，MTU=1500。实际上，这个最大传输单元常被限制为MTU-46，这是因为当IP层使用IP进行数据传输的时候，往往会将上层传下来的数据进行分割，然后按照IP头+数据的格式，向下发送。因此，为了保证上下层之间传输的兼容性，上层的数据包大小必须受到一定的限制，而这个限制就是MTU。


其余的状态:
- CLOSED
  - 没有任何连接状态
- FIN_WAIT_1
  - 等待远程 TCP 的连接中断请求，或先前的连接中断请求的确认
- FIN_WAIT_2
  - 从远程 TCP 等待连接中断请求
- SYN_SENT
  - 已发送了 SYN 连接请求
- SYN_RCVD
  - 收到一个连接请求，同时也已发送了连接请求
- CLOSING
  - 等待远程 TCP 对连接中断的确认
- LAST_ACK
  - 等待原来的连接中断请求的确认（和 CLOSE_WAIT 相对）

主动端可能出现的状态: FIN_WAIT1, FIN_WAIT2, CLOSING, TIME_WAIT

被动端可能出现的状态: CLOSE_WAIT, LAST_ACK, TIME_WAIT

## TCP 状态转换

客户端的状态可以用如下的流程表示:

CLOSED -> SYN_SENT -> ESTABLISHED -> FIN_WAIT_1 -> FIN_WAIT_2 -> TIME_WAIT -> CLOSED

服务端的状态可以用如下的流程表示:

CLOSED -> LISTEN -> SYN_RCVD -> ESTABLISHED -> CLOSE_WAIT -> LAST_ACK -> CLOSED

## TCP 各种标志位

TCP的标志位每个TCP段都有一个目的，这是借助于TCP标志位选项来确定的，允许发送方或接收方指定哪些标志应该被使用，以便段被另一端正确处理。用的最广泛的标志是 SYN，ACK 和 FIN，用于建立连接，确认成功的段传输，最后终止连接。

- SYN: 简写为S，同步标志位，用于建立会话连接，同步序列号；
- ACK: 简写为.，确认标志位，对已接收的数据包进行确认；
- FIN: 简写为F，完成标志位，表示我已经没有数据要发送了，即将关闭连接；
- PSH: 简写为P，推送标志位，表示该数据包被对方接收后应立即交给上层应用，而不在缓冲区排队；
- RST: 简写为R，重置标志位，用于连接复位、拒绝错误和非法的数据包；
- URG: 简写为U，紧急标志位，表示数据包的紧急指针域有效，用来保证连接不被阻断，并督促中间设备尽快处理；

## TCP 三次握手四次挥手

### 三次握手

![TCP三次握手](assets/doc04/tcp-three-way-handshake.png)


### 四次挥手

![TCP四次挥手](assets/doc04/tcp-four-way-wavehand.png)


