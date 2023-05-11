首先看一下TCP三次握手，四次挥手的整个过程

![image-20220807235106707](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220807235106707.png)



## TCP握手的简要操作流程

1. client端发送一个SYN包，请求建立连接，并告知server的初始化序号为X
2. server端收到后，回复一个ACK
3. server发送SYN包，告知client自己的初始化序号为Y，这个SYN与步骤【2】中的ACK，合并为一个包发送到客户端
4. client收到服务端ACK后，回复server一个ACK



【注意】TCP中有个重要的原则，就是收到ACK包的一端，不会对收到的ACK包做出ACK确认



## TCP连接队列

在TCP进行三次握手的时候，Linux会维护两个连接队列

- 半连接队列，也叫syn队列
- 全连接队列，也叫accept队列

![](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/20230109151621.png)

不管是半连接还是全连接队列，都有最大长度限制，超过限制时，内核会直接丢弃新接收的包，或者返回RST包



### 半连接队列

```shell
netstat -natp | grep SYN_RECV | wc -l
```

我们只需要一直对服务端发送syn包，但是不回ack回应包，这样就会使得服务端有大量请求处于syn_recv状态，这就是所谓的syn洪泛，syn攻击，DDos攻击

#### 如何抵御syn攻击

1. 增大半连接队列

   不能只增大tcp_max_syn_backlog，还需要一同增大somaconn和backlog，也就是增大全连接队列

2. 开启tcp_syncookies功能

   开启tcp_syncookies就可以在不适用syn半连接队列的情况下建立连接

   syncookies在接收到客户端的syn报文时，计算出一个值，放到syn+ack报文中发出。当客户端返回ack报文，取出该值验证，成功则建立连接，如图

   ![](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/20230109154151.png)

3. 减少ack+syn报文的重传次数

​		因为我们在收到syn攻击时，服务端会重传syn+ack报文到最大次数，才会断开连接。针对syn攻击的场景，我们可以减少ack+syn报文的重传次数，使处于syn_recv状态的它们更快断开连接

`修改重传次数:/proc/sys/net/ipv4/tcp_synack_retries`



### 全连接队列

当服务端的全连接队列过小时，容易发生全连接队列溢出。当全连接队列溢出时，后续的请求就会被丢弃

![](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/20230109164702.png)

`Linux有个参数可以指定TCP全连接队列满了，会使用什么策略来回应客户端`

丢弃连接只是 Linux 的默认行为，我们还可以向客户端发送RST报文终止连接，告诉客户端连接失败

![image-20230109170308632](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20230109170308632.png)

#### 如何增加全连接队列长度

`TCP 全连接队列的最大值取决于 somaxconn 和 backlog 之间的最小值，也就是 min(somaxconn, backlog)`，所以我们需要提高这两个参数的大小才能拿增大全连接队列

这个backlog，是调用listen函数传入的第二个参数

参考文献：https://blog.csdn.net/small_engineer/article/details/124190620





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

### 5-2 TIME_WAIT 用于解决什么问题

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



## 6.清除TIME_WAIT的技巧

可以用下面两种方式控制服务器 TIME_WAIT 的数量

【1】 修改 TCP_MAX_TW_BUCKETS 

​	TCP_MAX_TW_BUCKETS 控制并发的 TIME_WAIT 数量，默认是180000。如果超过默认值，内核会把多的TIME_WAIT连接清掉，并在日志里打一个警告。

这个选项只是为了阻止一些简单的DDos攻击， 平常不要人为的降低

【2】利用 RST 包从外部清掉 TIME_WAIT 连接。

​	假设 client 与 server 进行通信，且server主动关闭连接并进入 TIME_WAIT.

​	根据 IP_TRANSPARENT 这个socket，另起一个代理服务器，来模拟客户端给服务端发送一个FIN，这时处于TIME_WAIT状态下的Server是会作出ACK确认的。这个ACK确认最后会到达已经断开连接的client，但此时client已经离开了，所以会对这个ACK信号返回一个RST。Server收到这个RST后，就会提前终止TIME_WAIT状态

当然，提前终止TIME_WAIT状态，还是有机会出现 【5-2】中的3个问题，所以其实并不建议过早结束TIME_WAIT状态

这个过程，其实就是RST攻击

根据 TCP 规范，收到任何的发送到未侦听端口、已经关闭的连接的数据包、连接处于任何非同步状态（LISTEN,SYS-SENT,SYN-RECEIVED）并且收到的包的 ACK 在窗口外，或者安全层不匹配，都要回执以 RST 响应(而收到滑动窗口外的序列号的数据包，都要丢弃这个数据包，并回复一个 ACK 包)，内核收到 RST 将会产生一个错误并终止该连接。



## 7. TCP的延迟确认机制

按照TCP协议，确认机制是累积的。也就是说，确认号X的确认，指示的是所有X之前但不包括X的数据已经收到了。确认号（ACK）本身就是不含数据的分段。因此大量的确认号消耗了大量的带宽。虽然大多数情况下，ACK还是可以和数据一起捎带传输的，但是如果没有捎带传输，那么就只能单独传输一个ACK。如果这样的分段太多，网络的利用率就会下降。为了缓解这个问题，RFC建议了一种延迟的ACK，也就是说，在收到数据后，并不马上做出ACK，而是延迟一段可以接受的时间。

**延迟ACK的目的是看能不能和接收方要发送给发送方的数据一起回去，因为TCP协议投总是包含确认号的，如果可以的话，就连同ACK和数据一起捎带回去**

延迟ACK就算没有数据捎带，那么如果收到了按序的两个包，那么只要对第二包做确认即可，这样也可以节省一个ACK。由于TCP不对ACK进行ACK，RFC建议最多等待两个包的积累确认。

Linux实现中，有延迟ACK和快速ACK，并根据当前的包的收发情况在这两种ACK策略下切换。一般情况下，ACK不会对网络性能有太大的影响，

- 延迟ACK能减少发送的分段从而减少带宽
- 快速ACK能及时通知发送方丢包，避免滑动窗口停，提升吞吐率

关于 ACK 分段，有个细节需要说明一下，ACK 的确认号，是确认按序收到的最后一个字节序，对于乱序到来的 TCP 分段，接收端会回复相同的 ACK 分段，只确认按序到达的最后一个 TCP 分段。TCP 连接的延迟确认时间一般初始化为最小值 40ms，随后根据连接的重传超时时间（RTO）、上次收到数据包的时间间隔等参数进行不断调整



## 8. TCP的重传机制以及重传的超时计算

### 【8-1】TCP的重传超时计算

TCP交互过程中，如果发送的包一直没有收到ACK确认，是否一直等待？等待多久进行重传？这个等待重传的时间，称为 RTO （Retransmission TimeOut）

对于这个重传时间的计算，显然不能去猜测，应该是通过一个算法去计算，且这个超时时间是随着网络的状态在变化的。

在RFC中定义了一个经典算法

```
[1] 首先采样计算RTT值（一个数据包从发出去到回来的时间，Round Trip Time）
[2] 然后计算平滑的RTT，称为Smoothed Round Trip Time (SRTT)，SRTT = ( ALPHA * SRTT ) + ((1-ALPHA) * RTT)
[3] RTO = min[UBOUND,max[LBOUND,(BETA*SRTT)]]
```



## 9.TCP的流程控制

一般来说，我们总希望数据传输的更快一些。但是如果发送方把数据发得过快，接收方就可能来不及接收，这就会造成数据的丢失。**流量控制（flow control**就是让发送方的发送速率不要太快，要让接收方来得及接收。但是这样一个需求看似简单，但却有如下一些小细节的问题

- 发送端如何做到比较方便的指导自己哪些包可以发送，哪些包不能发送
- 如果接收端通知一个零窗口给发送端，这个时候发送端还能不能继续发送数据。若不发送，那就一直等待接收端扣通知一个非0窗口吗？
- 如果接收端处理能力很慢，这样接收端的窗口很快就被填满，然后接收处理完的几个字节，发送方又马上发送几个字节？这样做是不是效率太低下，是否应该做一个发送缓存或者接收缓存



### 【9-1】

​	一个简洁的方案就是按照接收方的窗口通告，发送方维护一个一样大小的发送窗口即可。在窗口内的包可以发送，窗口外的不能发。窗口在发送序列上不断后移，这个就是TCP中的滑动窗口。

![image-20220816110155574](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220816110155574.png)



滑动窗口将发送缓存内的数据分为4类：

- 已经发送，并得到接收端ACK的
- 已经发送，但未收到接收端ACK的
- 未发送但允许发送的（接收方还有空间）
- 未发送且不允许发送的（接收方没有恐龙）

其中【2】和【3】两部分合起来称为 **发送窗口**

![image-20220816112939789](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220816112939789.png)

### 【9-2】

发送端的发送窗口是由接收端来控制的，如下图:

![image-20220816113207694](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220816113207694.png)

由图可知，当接收端通知一个zero窗口的时候，发送端的发送窗口也变成0，也就是发送端不能发数据了。如果发送端一直等待，直到接收端通知一个非0窗口在发数据的话，这似乎就太受限于接收端。如果接收端一直不通知新的窗口呢？显然发送端也不能一直干等，起码需要有一个主动探测的机制。

为解决0窗口的问题，TCP使用了 Zero Window Probe 技术，缩写为 ZWP。发送端在窗口变成0后，会发 ZWP 的包给接收方，来探测接收端的窗口大小。一般会发送三次，每次大约30-60s



如果3次过后还是0的话，有的TCP实现就会发RST来中断这个链接。但这个 ZWP，可以被有心人利用，并对服务器发出攻击

攻击者可以在和 Server 建立好连接后，就向 Server 通告一个 0 窗口，然后 Server 端就只能等待进行 ZWP，于是攻击者会并发大量的这样的请求，把 Server 端的资源耗尽。

### 【9-3】

本质就是一个避免发送大量小包的问题。造成这个问题的原因有两个

- 接收端一直在通知一个小的窗口
- 发送端本身的问题，一直在发送小包。这个问题，在TCP中有个属于较 Silly Window Syndrome（糊涂窗口综合征）

对应的，解决这个问题的思路有两个

- 接收端不通知小窗口
- 发送端积累一下数据再发送

#### 【思路一】

​		在接收端解决这个问题，如果收到的数据导致window size 小于某个值，就ACK一个0的窗口，这就组着发送端再发数据过来。等到接收端处理了一些数据后， window size 大于等于 MSS（Maximum Segment Size 最大报文段长度），或者buffer有一半为空，就可以通告一个非0窗口。

#### 【思路二】

​	在发送端解决这个问题。就是大名大名鼎鼎的Nagle算法。Nagle算法主要是避免发送小的数据包，要求**TCP连接上最多只能有一个未被确认的小分组，在该分组的确认到达之前不能发送其他的小分组**。相反，TCP收集这些少量的小分组，并在确认到来时以一个分组的方式发出去。有如下规则

- 如果包长度达到MSS，则允许发送
- 如果该包含有FIN，则允许发送
- 设置了 TCP_NODELAY 选项，则允许发送
- 设置 TCP_CORK 选项，若所有发出去的小数据（包长度小于 MSS）均被确认，则允许发送
- 上述条件都未满足，但发送了超时（一般为200ms），则立即发送

TCP_CORK 选项则是禁止发送小的数据包(超时时间内)，设置该选项后，TCP 会尽力把小数据包拼接成一个大的数据包（一个 MTU）再发送出去，当然也不会一直等，发生了超时（一般为 200ms），也立即发送。Nagle 算法和 CP_CORK 选项提高了网络的利用率，但是增加是延时。

```
若发送应用进程把要发送的数据逐个字节地送到TCP的发送缓存，则发送方就把第一个数据字节先发送出去，把后面的数据字节都缓存起来。当发送方收到对第一个数据字符的确认后，再把发送缓存中的所有数据组装成一个报文段发送出去，同时继续对后面的数据先进行写入缓存。只有收到对签名一个报文段的确认后才继续发送下一段报文。
```

Nagle算法与延迟确认，可能会导致一个40ms的延时问题

![image-20220816145052497](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20220816145052497.png)

## 10. TCP拥塞控制

本质上，网络上拥塞的原因就是大家都想独享整个网络资源，对于TCP，端到端的流量控制必然会导致网络拥堵。这是因为TCP只看到对端的接收空间大小，而无法知道链路上的容量。只要双方的处理能力很强，那么就可以以很大的速率发包，于是链路很快出现拥堵，进而引起大量的丢包，丢包有引发发送端的重传风暴，进一步加剧链路的拥塞。

另外一个拥塞的因素是链路上的转发节点，例如路由器，再好的路由器只要接入网络，总是会拉低网络的总带宽，如果在路由器节点上出现处理瓶颈，那么久很容易出现拥塞。

拥塞控制要完成两个任务：

- 公平性
- 拥塞过后的恢复

TCP发展都现在，拥塞控制方面的算法很多，其中Reno是目前应用最广泛且较为成熟的算法。

慢启动、拥塞避免、快重传、快恢复

在 TCP协议.md 中有详细过程，不再赘述





## 参考文献

- https://blog.csdn.net/qq1263575666/article/details/123017054
- https://blog.csdn.net/small_engineer/article/details/124190620
- https://blog.csdn.net/dog250/article/details/7518054/

- https://zhensheng.im/tag/ip_transparent 透明反向代理实现