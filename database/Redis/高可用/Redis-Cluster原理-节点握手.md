## 搭建概述

Cluster搭建一般有三个步骤

- 节点准备，redis.conf文件配置
- 节点握手
- slot分配

Cluster工作要点

- key分配到槽点
- rehash，slot重分配
- 故障转移（主从切换）
- 节点下线



以下根据这几点进行介绍，但是在介绍前，先介绍下Gossip协议。

## Goosip协议

Redis Cluster 各个节点之间交换数据、通信采用的的消息协议，叫做 Gossip协议，即流言协议

![image-20210205111135027](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210205111135027.png)

Redis节点每秒钟向其他节点发送PING，然后被PING的节点响应PONG。

在Redis-Cluster里面，节点间的消息类型有5种，分别是MEET、PING、PONG、FAIL和PUBLISH。

| 消息类型 | 消息内容                                                     |
| -------- | ------------------------------------------------------------ |
| MEET     | 给某个节点发送MEET消息，请求接收消息的节点加入到集群中       |
| PING     | 每隔一秒钟，选择5个最久没有通信的节点，发送PING消息，检测对应的节点是否在线；同时还有一种策略是，如果某个节点的通信延迟大于了`cluster-node-time`的值的一半，就会立即给该节点发送PING消息，避免数据交换延迟过久 |
| PONG     | 当节点接收到MEET或者PING消息之后，会回一个PONG消息给发送方，代表自己收到了MEET或者PING消息。同时，节点也可以主动的通过PONG消息向集群中广播自己的信息，让其他节点获取到自己最新的属性，就像完成了故障转移之后新的master向集群发送PONG消息一样 |
| FAIL     | 用于广播自己的对某个节点的宕机判断，假设当前节点对A节点判断为宕机，就会立即向Redis Cluster广播自己对于A节点的判断，所有收到消息的节点就会对A节点做标记 |
| PUBLISH  | 用于向指定的Channel发送消息，某个节点收到PUBLISH消息之后会直接在集群内广播，这样一来，客户端无论连接到任何节点都能够订阅这个Channel |

既然Redis Cluster选择了gossip，那肯定存在一些gossip的优点，我们接下来简单梳理一下。

| 优点     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| 扩展性   | 网络可以允许节点的任意增加和减少，新增加的节点的状态最终会与其他节点一致。 |
| 容错性   | 由于每个节点都持有一份完整元数据，所以任何节点宕机都不会影响gossip的运行 |
| 健壮性   | 与容错性类似，由于所有节点都持有数据，地位平台，是一个去中心化的设计，任何节点都不会影响到服务的运行 |
| 终一致性 | 当有新的信息需要传递时，消息可以快速的发送到所有的节点，让所有的节点都拥有最新的数据 |

gossip可以在**O(logN) 轮**就可以将信息传播到所有的节点，为什么是O(logN)呢？因为每次ping，当前节点会带上自己的信息外加整个Cluster的1/10数量的节点信息，一起发送出去。你可以简单的把这个模型抽象为：

> 你转发了一个特别有意思的文章到朋友圈，然后你的朋友们都觉得还不错，于是就一传十、十传百这样的散播出去了，这就是朋友圈的**裂变传播**。

当然，gossip仍然存在一些缺点。例如消息可能最终会经过很多轮才能到达目标节点，而这可能会带来较大的延迟。同时由于节点会随机选出5个最久没有通信的节点，这可能会造成某一个节点同时收到n个重复的消息。



所有的消息格式划分为：消息头和消息体

- 消息头包含了发送节点自身状态数据，接收节点根据消息头就可以获取到发送节点的相关数据。集群内所有的消息都采用相同的消息头结构`clusterMsg`

```c
# 结构内包含了关键信息，如节点Id，槽映射，节点标识（主从角色、是否下线）
typedef struct {
    char sig[4]; /* 信号标示 */
    uint32_t totlen; /* 消息总长度 */
    uint16_t ver; /* 协议版本*/
    uint16_t type; /* 消息类型,用于区分meet,ping,pong等消息 */
    uint16_t count; /* 消息体包含的节点数量，仅用于meet,ping,ping消息类型*/
    uint64_t currentEpoch; /* 当前发送节点的配置纪元 */
    uint64_t configEpoch; /* 主节点/从节点的主节点配置纪元 */
    uint64_t offset; /* 复制偏移量 */
    char sender[CLUSTER_NAMELEN]; /* 发送节点的nodeId */
    unsigned char myslots[CLUSTER_SLOTS/8]; /* 发送节点负责的槽信息 */
    char slaveof[CLUSTER_NAMELEN]; /* 如果发送节点是从节点，记录对应主节点的nodeId */
    uint16_t port; /* 端口号 */
    uint16_t flags; /* 发送节点标识,区分主从角色，是否下线等 */
    unsigned char state; /* 发送节点所处的集群状态 */
    unsigned char mflags[3]; /* 消息标识 */
    union clusterMsgData data /* 消息正文 */;
} clusterMsg;
```

- 消息体在Redis内部采用clusterMsgData结构声明，如下

```c
union clusterMsgData {
    /* ping,meet,pong消息体*/
    struct {
        /* gossip消息结构数组 */
        clusterMsgDataGossip gossip[1];
    } ping;
    /* FAIL 消息体 */
    struct {
        clusterMsgDataFail about;
    } fail;
    // ...
};
```

消息体clusterMsgData定义发送消息的数据，**其中ping、meet、pong都采用cluster MsgDataGossip数组作为消息体数据**，实际消息类型使用消息头的type属性区分。每个消息体包含该节点的多个clusterMsgDataGossip结构数据，用于信息交换，如下：

```c
typedef struct {
    char nodename[CLUSTER_NAMELEN]; /* 节点的nodeId */
    uint32_t ping_sent; /* 最后一次向该节点发送ping消息时间 */
    uint32_t pong_received; /* 最后一次接收该节点pong消息时间 */
    char ip[NET_IP_STR_LEN]; /* IP */
    uint16_t port; /* port*/
    uint16_t flags; /* 该节点标识, */
} clusterMsgDataGossip;
```

Cluster节点间的通讯，全都是围绕着这个Gossip协议进行的。



## Cluster节点握手

节点握手后，才能进行信息的交流，有两种情况会触发节点的握手

- 执行命令 `cluster meet {ip} {port}`

- 接收到PING命令后，解析消息体中的clusterMsgDataGossip中包含的节点是否为新节点，若为新节点，则尝试与新节点进行 meet 握手流程。

第一种情况是在客户端直接发起与指定节点的握手，第二种是节点信息传递时，自动发现新节点。

第二种情况发生的前提是，在已握手的节点里面，至少有一个节点已经与新节点进行握手，即至少执行了一次 `cluster meet`命令。



但是这两种情况，归根结底都是执行了 `cluster meet`命令。下面详细介绍这里命令的底层原理。

这里涉及到三个结构：clusterNode，clusterLink，clusterState

## clusterNode

该结构保存了节点的当前状态信息，如节点的创建时间，节点的名字(节点的运行ID)，节点当前的配置纪元，节点IP和端口信息等等。

**每一个节点都会使用一个clusterNode结构来记录自己的状态**，并为集群中的所有其他节点（包括主节点和从节点）都创建一个相应的clusterNode结构，以此来记录其他节点的状态。

```c
struct clusterNode {
    //创建节点的时间
    mstime_t ctime;
 
    //节点的名字，由40 个十六进制字符组成
    //例如68eef66df23420a5862208ef5b1a7005b806f2ff
    char name[REDIS_CLUSTER_NAMELEN];
 
    //节点标识
    //使用各种不同的标识值记录节点的角色（比如主节点或者从节点），
    //以及节点目前所处的状态（比如在线或者下线）。
    int flags;
 
    //节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;
 
    //节点的IP 地址
    char ip[REDIS_IP_STR_LEN];
 
    //节点的端口号
    int port;
 
    //保存连接节点所需的有关信息
    clusterLink *link;
    
    //记录当前节点处理的slot，若索引i上的二进制位的值为1，那么表示节点负责处理槽i
    unsigned char slots[16384/8];
    
    //记录该节点负责处理的槽的数量
    int numslots;
};
```



## ClusterLinks

clusterNode结构的link属性是一个clusterLink结构，该结构保存了连接节点所需要的有关信息。比如套接字描述符，输入缓冲区和输出缓冲区。

```c
typedef struct clusterLink {
    //连接的创建时间
    mstime_t ctime;
 
    // TCP 套接字描述符
    int fd;
 
    //输出缓冲区，保存着等待发送给其他节点的消息（message ）。
    sds sndbuf;
 
    //输入缓冲区，保存着从其他节点接收到的消息。
    sds rcvbuf;
 
    //与这个连接相关联的节点，如果没有的话就为NULL
    struct clusterNode *node;
} clusterLink;
```

**redisClient结构和clusterLink结构的相同和不同之处：**

- redisClient结构和clusterLink结构都有自己的套接字描述符和输入、输出缓冲区，这两个结构的区别在于，**redisClient结构中的套接字和缓冲区是用于连接客户端的**，而**clusterLink结构中的套接字和缓冲区则是用于连接节点的**

## clusterState

每个节点都保存一个clusterState结构，这个结构记录了在当前节点的视角下，集群目前所处的状态。如集群是在线还是下线，集群包含了多少个节点，集群当前的配置纪元。

```c
typedef struct clusterState {
    // 指向当前节点的指针
    clusterNode *myself;
 
    // 集群当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;
 
    // 集群当前的状态：是在线还是下线
    int state;
 
    // 集群中至少处理着一个槽的节点的数量
    int size;
 
    //集群节点名单（包括myself 节点）
    //字典的键为节点的名字（运行ID），字典的值为节点对应的clusterNode 结构
    dict *nodes;
    
    //包含集群中所有slot的指派信息，每个数组项都是一个指向clusterNode结构的指针
    clusterNode *slots[16384];
    
} clusterState;
```

![image-20210205163740673](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210205163740673.png)



- 结构的**currentEpoch属性的值为0**，表示集群当前的配置纪元为0
- 结构的**size属性的值为0**，表示集群目前没有任何节点在处理槽，因此结构的state属性的 值为REDIS_CLUSTER_FAIL，这表示集群目前处于下线状态
- 结构的**nodes字典记录了集群目前包含的三个节点**，这三个节点分别由三个clusterNode 结构表示，其中my self指针指向代表节点7000的clusterNode结构，而字典中的另外两个指针 则分别指向代表节点7001和代表节点7002的clusterNode结构，这两个节点是节点7000已知的 在集群中的其他节点
- **三个节点的clusterNode结构的flags属性**都是REDIS_NODE_MASTER，说明三个节点都是主节点



节点7001 、7002中也会创建类似的clusterState结构。

- 不过在节点7001创建的clusterState结构中，my self指针将指向代表节点7001的 clusterNode结构，而节点7000和节点7002则是集群中的其他节点
- 而在节点7002创建的clusterState结构中，my self指针将指向代表节点7002的clusterNode 结构，而节点7000和节点7001则是集群中的其他节点



## CLUSTER MEET命令实现

节点握手的本质，就是`cluster meet {ip} {port}`命令的实现

通过向节点A发送CLUSTER MEET命令，客户端**可以让接收命令的节点A将另一个节点B添加到节点A当前所在的集群里面：**

```c
CLUSTER MEET <ip> <port>
```

收到命令的节点A将与节点B进行握手（handshake），**以此来确认彼此的存在，并为将来的进一步通信打好基础：**

- ①节点A会为节点B创建一个clusterNode结构，并将该结构添加到自己的 clusterState.nodes字典里面
- ②之后，节点A将根据CLUSTER MEET命令给定的IP地址和端口号，向节点B发送一条 MEET消息（message）
- ③如果一切顺利，节点B将接收到节点A发送的MEET消息，节点B会为节点A创建一个 clusterNode结构，并将该结构添加到自己的clusterState.nodes字典里面
- ④之后，节点B将向节点A返回一条PONG消息
- ⑤如果一切顺利，节点A将接收到节点B返回的PONG消息，通过这条PONG消息节点A 可以知道节点B已经成功地接收到了自己发送的MEET消息
- ⑥之后，节点A将向节点B返回一条PING消息
- ⑦如果一切顺利，节点B将接收到节点A返回的PING消息，通过这条PING消息节点B可以知道节点A已经成功地接收到了自己返回的PONG消息，握手完成



![image-20210205170552323](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210205170552323.png)

当A与B建立连接后，节点A会将节点B的信息通过gossip协议传播给集群中的其他节点，让其他节点也与节点B进行握手，最终经过一段时间，节点B被所有节点认识。



cluster meet命令的本质就是发送一条消息类型为meet的gossip消息。

节点每次发送gossip消息时，都会在消息体 clusterMsgData 中的 clusterMsgDataGossip中，携带节点信息。接收节点信息就是通过这个clusterMsgDataGossip里面的信息来与自己的clusterState 里面的node字典进行判断，若果是新节点，则发送一条meet消息与新节点进行握手。否则就更新这个节点的信息，如槽映射信息，主从角色等等。



而其中的关键之处，就是发送gossip信息时，如何选择clusterMsgDataGossip里面应该携带哪个节点，向哪个节点发送数据。这个就是节点的选择问题。



## 节点选择

gossip协议的信息交换机制具有天然的分布式特性，但是这个是具有带宽成本的。如果内部需要频繁的进行节点信息交换，而**ping/pong消息会携带当前节点和部分节点的状态数据，必然会增加带宽和计算的负担**。 

Redis集群内部节点通信采用固定频率（定时任务每秒执行10次）。换言之就是通信节点的选择需要非常严谨。

通讯节点过多，可以做到信息及时交换，但是成本过高；节点过少会降低集群内所有节点的信息交换频率，直接影响故障判定、新节点发现的速度等等。



考虑到通讯的实时性和成本开销，Redis-Cluster的通讯节点规则选择如下：

![image-20210205180510223](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210205180510223.png)



### 选择发送的对象

有两种对象，这是从一次定时任务调用中完成的，每100ms运行一次。

1. 先随机挑选5个节点，从中挑选出最近一次接收 PONG 回复距离现在最旧的节点
2. 每100毫秒都会扫描本地节点列表，如果发现节点最近一次接受pong 消息的时间大于cluster_node_timeout/2，则立刻发送ping消息，防止该节点信 息太长时间未更新。

### 选择携带的信息

```c
/* Send a PING or PONG packet to the specified node, making sure to add enough
 * gossip informations. */
// 向指定节点发送一条 MEET 、 PING 或者 PONG 消息
void clusterSendPing(clusterLink *link, int type) {
    unsigned char buf[sizeof(clusterMsg)];
    clusterMsg *hdr = (clusterMsg*) buf;
    int gossipcount = 0, totlen;
    /* freshnodes is the number of nodes we can still use to populate the
     * gossip section of the ping packet. Basically we start with the nodes
     * we have in memory minus two (ourself and the node we are sending the
     * message to). Every time we add a node we decrement the counter, so when
     * it will drop to <= zero we know there is no more gossip info we can
     * send. */
    // freshnodes 是用于发送 gossip 信息的计数器
    // 每次发送一条信息时，程序将 freshnodes 的值减一
    // 当 freshnodes 的数值小于等于 0 时，程序停止发送 gossip 信息
    // freshnodes 的数量是节点目前的 nodes 表中的节点数量减去 2 
    // 这里的 2 指两个节点，一个是 myself 节点（也即是发送信息的这个节点）
    // 另一个是接受 gossip 信息的节点
    int freshnodes = dictSize(server.cluster->nodes)-2;
 
    // 如果发送的信息是 PING ，那么更新最后一次发送 PING 命令的时间戳
    if (link->node && type == CLUSTERMSG_TYPE_PING)
        link->node->ping_sent = mstime();
 
    // 将当前节点的信息（比如名字、地址、端口号、负责处理的槽）记录到消息里面
    clusterBuildMessageHdr(hdr,type);
 
    /* Populate the gossip fields */
    // 从当前节点已知的节点中随机选出两个节点
    // 并通过这条消息捎带给目标节点，从而实现 gossip 协议
 
    // 每个节点有 freshnodes 次发送 gossip 信息的机会
    // 每次向目标节点发送 2 个被选中节点的 gossip 信息（gossipcount 计数）
    while(freshnodes > 0 && gossipcount < 3) {
        // 从 nodes 字典中随机选出一个节点（被选中节点）
        dictEntry *de = dictGetRandomKey(server.cluster->nodes);
        clusterNode *this = dictGetVal(de);
 
        clusterMsgDataGossip *gossip;
        int j;
 
        /* In the gossip section don't include:
         * 以下节点不能作为被选中节点：
         * 1) Myself.
         *    节点本身。
         * 2) Nodes in HANDSHAKE state.
         *    处于 HANDSHAKE 状态的节点。
         * 3) Nodes with the NOADDR flag set.
         *    带有 NOADDR 标识的节点
         * 4) Disconnected nodes if they don't have configured slots.
         *    因为不处理任何槽而被断开连接的节点 
         */
        if (this == myself ||
            this->flags & (REDIS_NODE_HANDSHAKE|REDIS_NODE_NOADDR) ||
            (this->link == NULL && this->numslots == 0))
        {
                freshnodes--; /* otherwise we may loop forever. */
                continue;
        }
 
        /* Check if we already added this node */
        // 检查被选中节点是否已经在 hdr->data.ping.gossip 数组里面
        // 如果是的话说明这个节点之前已经被选中了
        // 不要再选中它（否则就会出现重复）
        for (j = 0; j < gossipcount; j++) {
            if (memcmp(hdr->data.ping.gossip[j].nodename,this->name,
                    REDIS_CLUSTER_NAMELEN) == 0) break;
        }
        if (j != gossipcount) continue;
 
        /* Add it */
 
        // 这个被选中节点有效，计数器减一
        freshnodes--;
 
        // 指向 gossip 信息结构
        gossip = &(hdr->data.ping.gossip[gossipcount]);
 
        // 将被选中节点的名字记录到 gossip 信息
        memcpy(gossip->nodename,this->name,REDIS_CLUSTER_NAMELEN);
        // 将被选中节点的 PING 命令发送时间戳记录到 gossip 信息
        gossip->ping_sent = htonl(this->ping_sent);
        // 将被选中节点的 PING 命令回复的时间戳记录到 gossip 信息
        gossip->pong_received = htonl(this->pong_received);
        // 将被选中节点的 IP 记录到 gossip 信息
        memcpy(gossip->ip,this->ip,sizeof(this->ip));
        // 将被选中节点的端口号记录到 gossip 信息
        gossip->port = htons(this->port);
        // 将被选中节点的标识值记录到 gossip 信息
        gossip->flags = htons(this->flags);
 
        // 这个被选中节点有效，计数器增一
        gossipcount++;
    }
 
    // 计算信息长度
    totlen = sizeof(clusterMsg)-sizeof(union clusterMsgData);
    totlen += (sizeof(clusterMsgDataGossip)*gossipcount);
    // 将被选中节点的数量（gossip 信息中包含了多少个节点的信息）
    // 记录在 count 属性里面
    hdr->count = htons(gossipcount);
    // 将信息的长度记录到信息里面
    hdr->totlen = htonl(totlen);
 
    // 发送信息
    clusterSendMessage(link,buf,totlen);
}
```



总结来说就是：

- 每次发送信息时，都会携带两个节点的信息
- 这两个节点信息是从当前节点中 nodes 字典中随机选择的
- 有几种情况，节点不能被选择作为携带对象
  - 自身节点
  - 处于 HANDSHAKE 状态的节点。
  - 带有 NOADDR 标识的节点
  - 因为不处理任何槽而被断开连接的节点 



注意点：

1. 这个每秒10次的调用，是在serverCron，即时间事件中进行的（这个每秒10次，是写死的，并不能通过配置修改）
2. 每个节点都有 freshnodes 次发送gossip的机会，这个freshnodes的数量是当前节点 nodes字典数量 -2 ，这个2，一个代表自己，一个代表接收信息的节点
3. 当一次发送消息事件里面，选取携带对象次数多于freshnodes 时，停止发送消息



```c
/* Run the Redis Cluster cron. */
// 如果服务器运行在集群模式下，那么执行集群操作
run_with_period(100) {
    if (server.cluster_enabled)     clusterCron();
}
```





## 参考文献

https://blog.csdn.net/hguisu/article/details/82979320

https://blog.csdn.net/weixin_30871905/article/details/95647441?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.control

https://blog.csdn.net/weixin_42667608/article/details/111360617

https://www.cnblogs.com/williamjie/p/11132211.html

https://blog.csdn.net/qq_41453285/article/details/103402727

https://blog.csdn.net/qq_41453285/article/details/103402608

https://blog.csdn.net/qq_41453285/article/details/103394429

https://blog.csdn.net/qq_41453285/article/details/103395224



https://littlemagic.blog.csdn.net/article/details/106330048?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1.control



关键词，Redis(设计与实现):51 江南、懂少

