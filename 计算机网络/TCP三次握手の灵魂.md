首先看一下TCP三次握手，四次挥手的整个过程

![image-20220807235106707](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220807235106707.png)



## TCP握手的简要操作流程

1. client端发送一个SYN包，请求建立连接，并告知server的初始化序号为X
2. server端收到后，回复一个ACK
3. server发送SYN包，告知client自己的初始化序号为Y，这个SYN与步骤【2】中的ACK，合并为一个包发送到客户端
4. client收到服务端ACK后，回复server一个ACK



【注意】TCP中有个重要的原则，就是收到ACK包的一端，不会对收到的ACK包做出ACK确认

## 1.TCP的初始化序列号是否可以固定

结论：不能

假设初始化序列（Initial Sequence Number）号是可以固定的，并且为1

现在client与server建立了一个tcp连接，且client发送了10个包，现在这10个包被链路上的路由器缓存起来了，然后这时候client挂了。然后 Client 用同样的端口号重新连上 Server，Client 又连续给 Server 发了几个包，假设这个时候 Client 的序列号变成了 5。

接着此时被路由器缓存的数据包，被服务端收到，server给client回复确认号10，这时收到这个确认包的client整个数据都会乱掉。

RFC793 中，建议 ISN 和一个假的时钟绑在一起，这个时钟会在每 4 微秒对 ISN 做加一操作，直到超过 2^32，又从 0 开始，这需要 4 小时才会产生 ISN 的回绕问题，这几乎可以保证每个新连接的 ISN 不会和旧的连接的 ISN 产生冲突。这种递增方式的 ISN，很容易让攻击者猜测到 TCP 连接的 ISN，现在的实现大多是在一个基准值的基础上进行随机的。

## 2.SYN超时，半连接队列溢出

Client 发送 SYN 包给 Server 后立刻挂了，无法对服务端的SYN包发送确认，若长时间没有给 Server 发送连接确认，这样就会占用SYN队列中的一个位置。

这就需要一个超时时间让 Server 将这个连接断开，否则这个连接就会一直占用 Server 的 SYN 连接队列中的一个位置，大量这样的连接就会将 Server 的 SYN 连接队列耗尽，让正常的连接无法得到处理。目前，Linux 下默认会进行 5 次重发 SYN-ACK 包，重试的间隔时间从 1s 开始，下次的重试间隔时间是前一次的双倍，5 次的重试时间间隔为 1s,2s, 4s, 8s,16s，总共 31s，第 5 次发出后还要等 32s 都知道第 5 次也超时了，所以，总共需要 1s + 2s +4s+ 8s+ 16s + 32s =63s，TCP 才会把断开这个连接。

由于SYN的超时需要整整63s，如果发起连接的client足够多，那么是可以把server端的SYN队列完全挤满，导致server无法再建立新的链接。这个叫就SYN攻击（ddos攻击）

linux 提供了几个 TCP 参数来调整应对。

- tcp_syncookies
- tcp_synack_retries
- tcp_max_syn_backlog
- tcp_abort_on_overflow 



【参考文献】

- 半连接与全连接队列 https://blog.csdn.net/small_engineer/article/details/124190620

## 3.TCP的peer两端同时发出FIN包

正常状态下：

1. 先发出FIN包的一端，变成 FIN_WAIT1,
2. 在FIN_WAIT1时，收到对端ACK确认时，变成FIN_WAIT2
3. 在FIN_WAIT2时，收到对端的FIN包，响应一个ACK包后，进入TIME_WAIT状态

但如果两端同时发出FIN包，则会如下

![image-20220808004139670](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220808004139670.png)



## 4.是否可以用三次挥手替代四次挥手

结论：理论上是可以的

如果 Server 在收到 Client 的 FIN 包后，在也没数据需要发送给 Client 了，那么对 Client 的 ACK 包和 Server 自己的 FIN 包就可以合并成为一个包发送过去，这样四次挥手就可以变成三次了(似乎 linux 协议栈就是这样实现的)



## 5.TCP TIME_WAIT状态

TCP 主动关闭连接的那一方会最后进入 TIME_WAIT

### 5-1 Peer两端，哪一个端会进入TIME_WAIT状态

如果界定主动关闭方。是否主动关闭是由 FIN 包的先后决定的，就是在自己没收到对端 Peer 的 FIN 包之前自己发出了 FIN 包，那么自己就是主动关闭连接的那一方。

但如果对于问题3，则Peer双端都是主动关闭方，都会进入TIME_WAIT状态。那么为何主动关闭的一方进入TIME_WAIT？

- 主动方发送FIN
- 被动关闭方收到主动关闭方的 FIN 后发送该 FIN 的 ACK
- 被动关闭方发送FIN
- 主动关闭方收到被动关闭方的 FIN 后发送该 FIN 的 ACK，被动关闭方等待自己 FIN 的 ACK。

问题就在于第四点。根据TCP规范，不会对ACK进行ACK。如果主动关闭方不进入TIME_WAIT，当主动关闭方发送完ACK后就离开的话，这样是无法保证ACK包到达被动方的。

如果被动方没有收到自己FIN的ACK就不能关闭，接着会继续重发FIN包。但此时已经没有可以接收这个FIN包的Peer端了。这样被动方就一直不能关闭，这个链接的资源就不能释放。

**所以主动关闭方会进入TIME_WAIT状态，以便可以对被动关闭方的FIN包做出ACK**

### 5-2 TIME_WAIT的作用

- 主动关闭方需要进入 TIME_WAIT 以便能够重发丢掉的被动关闭方 FIN 包的 ACK。

- 防止已经断开的连接 1 中在链路中残留的 FIN 包终止掉新的连接 2(重用了连接 1 的所有的 5 元素(源 IP，目的 IP，TCP，源端口，目的端口)），这个概率比较低，因为涉及到一个匹配问题，迟到的 FIN 分段的序列号必须落在连接 2 的一方的期望序列号范围之内，虽然概率低，但是确实可能发生，因为初始序列号都是随机产生的，并且这个序列号是 32 位的，会回，而绕。

  经过TIME_WAIT状态后（TIME_WAIT状态会持续一段时间），这些残留于链路中的FIN包都会得到ACK

- 防止链路上已经关闭的连接的残余数据包(a lost duplicate packet or a wandering duplicate packet) 干扰正常的数据包，造成数据流的不正常。这个问题和 2）类似。

### 5-3 TIME_WAIT带来的问题

一个连接进入 TIME_WAIT 状态后需要等待 2*MSL(一般是 1 到 4 分钟)那么长的时间才能断开连接释放连接占用的资源，会造成以下问题：

- 作为服务器，短时间内关闭了大量的 Client 连接，就会造成服务器上出现大量的 TIME_WAIT 连接，占据大量的 tuple，严重消耗着服务器的资源。
- 作为客户端，短时间内大量的短连接，会大量消耗的 Client 机器的端口，毕竟端口只有 65535 个，端口被耗尽了，后续就无法在发起新的连接了。

### 5-4 TIME_WAIT的快速回收与重用

1. **TIME_WAIT 快速回收**

​	linux 下开启 TIME_WAIT 快速回收需要同时打开 tcp_tw_recycle 和 tcp_timestamps(默认打开)两选项。linux下快速回收的时间为3.5 * RTO（Retransmission Timeout），而一个RTO时间为200ms至120ms。开始快速回收TIME_WAIT，可能会带来 5-2 中的问题。所以要求同时满足以下三个条件的新连接就要被拒绝：

- 来自同一个对端 Peer 的 TCP 包携带了时间戳；
- 之前同一台 peer 机器(仅仅识别 IP 地址，因为连接被快速释放了，没了端口信息)的某个 TCP 数据在 MSL 秒之内到过本 Server；

- Peer 机器新连接的时间戳小于 peer 机器上次 TCP 到来时的时间戳，且差值大于重放窗口戳(TCP_PAWS_WINDOW)。

初看起来正常的数据包同时满足下面 3 条几乎不可能，因为机器的时间戳不可能倒流的，出现上述的 3 点均满足时，一定是老的重复数据包又回来了，丢弃老的 SYN 包是正常的。到此，似乎启用快速回收就能很大程度缓解 TIME_WAIT 带来的问题。但是，这里忽略了一个东西就是 NAT。



在一个 NAT 后面的所有 Peer 机器在 Server 看来都是一个机器，NAT 后面的那么多 Peer 机器的系统时间戳很可能不一致，有些快，有些慢。这样，在 Server 关闭了与系统时间戳快的 Client 的连接后，在这个连接进入快速回收的时候，同一 NAT 后面的系统时间戳慢的 Client 向 Server 发起连接，这就很有可能同时满足上面的三种情况，造成该连接被 Server 拒绝掉。所以，在是否开启 tcp_tw_recycle 需要慎重考虑了



2. TIME_WAIT重用

只要满足下面两点中的一个，一个TIME_WAIT状态的四元组可以被新到来的SYN重用

- 新连接 SYN 告知的初始序列号比 TIME_WAIT 老连接的末序列号大；
- 如果开启了 tcp_timestamps，并且新到来的连接的时间戳比老连接的时间戳大。

关于 tcp_tw_reuse 和 SO_REUSEADDR 选项，其实是两个完全不同的东西。

**tcp_tw_reuse 是内核选项，而 SO_REUSEADDR 用户态的选项**，使用 SO_REUSEADDR 是告诉内核，如果端口忙，但 TCP 状态位于 TIME_WAIT，可以重用端口。如果端口忙，而 TCP 状态位于其他状态，重用端口时依旧得到一个错误信息，指明 Address already in use”。 

如果你的服务程序停止后想立即重启，而新套接字依旧使用同一端口，则 SO_REUSEADDR 会非常有用



## 参考文献

https://blog.csdn.net/qq1263575666/article/details/123017054