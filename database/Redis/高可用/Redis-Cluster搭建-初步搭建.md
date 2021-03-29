# Cluster概述

随着业务增长，Redis需要承载更多QPS，【主从架构 + 哨兵集群 】中，受限于

- 主库写能力受单机限制
- 主库存储能力受单机限制
- COW写时复制，在极限情况下主库内存溢出
- 资源浪费，Redis数据节点Slave节点作为备份节点不提供服务

等等因素，主从 + 哨兵 已经不能满足业务要求。

Redis Cluster 是社区版推出的 Redis 分布式集群解决方案，主要解决 Redis 分布式方面的需求，比如，当遇到单机内存，并发和流量等瓶颈的时候，Redis Cluster 能起到很好的负载均衡的目的。



## Redis-Cluster结构

Cluster，集群的意思。RedisCluster，就是同时有多个主从结构向外提供服务。

**Redis-Cluster要求至少3个master才能组成一个集群，同时每个master至少需要一个slave节点**

![image-20210203172215262](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210203172215262.png)

主从架构中，可以通过增加slave节点的方式来扩展读请求的并发量，但是在Redis-Cluster中，虽然每个master下都挂载了一个slave节点，但是在Redis Cluster中的读、写请求其实都是在**master**上完成的。

**slave节点只是充当了一个数据备份的角色，当master发生了宕机，就会将对应的slave节点提拔为master，来重新对外提供服务。**



在集群中任何两个节点是相互连通的，客户端可以任一节点相连接，然后就可以访问集群中的任何一个节点，对其进行存取和其他操作。

- 由多个Redis服务器组成的分布式网络服务集群
- 集群之中有多个Master主节点，每个主节点都可读可写
- 节点之间会互相通信，两两相连
- Redis集群无中心节点

- 集群中故障转移由集群中其他在线的主节点负责，无须使用Sentinel



## 节点负载均衡

**在Redis-Cluster中，每个master里面存储的数据并不是一样的，每一个key，都是通过负载均衡计算之后，再决定存放到哪一个master里面。**

![image-20210203184705424](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210203184705424.png)



​		一般情况来说，负载均衡算法选取的是hash算法，首先就是对key计算出一个hash值，然后用哈希值对master数量进行取模。由此就可以将key负载均衡到每一个Redis节点上去。这就是简单的**哈希算法**的实现。

​		简单的哈希算法有个巨大的弊端。如果此时某一台master发生了宕机，那么此时会导致Redis中**所有的缓存失效**。假设之前有3个master，那么之前的算法应该是 hash % 3，但是如果其中一台master宕机了，则算法就会变成 hash % 2，会影响到之前存储的所有的key。而这对缓存后面保护的DB来说，是致命的打击。

**而在Redis-Cluster中，负载均衡采取的是类一致性哈希算法。**

首先来看下什么是一致性哈希算法

### 一致性哈希算法

普通的哈希算法是对master实例的数量进行取模操作，而一致性哈希是对2^32取模，也就是值的范围在 [0, 2^32 -1]，一致性哈希将其范围抽象成了一个圆环，使用CRC16算法计算哈希值会落到圆环的某个地方。

![image-20210203185633447](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210203185633447.png)

然后把master实例也分布在这个圆环上。

计算出Key的落点后，按照顺时针方向寻找，找到的第一个master实例，就把key存储在这个实例中。

假设我们有A、B、C三个Redis实例按照如图所示的位置分布在圆环上，此时计算出来的hash值，取模之后位置落在了**位置D**，那么我们按照顺时针的顺序，就能够找到我们这个key应该分配的Redis实例B。同理如果我们计算出来位置在E，那么对应选择的Redis的实例就是A。

### 虚拟节点机制

一致性哈希也存在不小的问题，当redis节点不是平均分布的时候，整个集群里的数据分布也不平衡，如下图

![image-20210203222446826](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210203222446826.png)



数据落在节点A上的概率明显是大于其他两个节点的，其次落在节点C上的概率最小。这样一来会导致整个集群的数据存储不平衡，AB节点压力较大，而C节点资源利用不充分。为了解决这个问题，一致性哈希算法引入了**虚拟节点机制**。

![image-20210203225547078](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210203225547078.png)

在圆环中，增加了对应节点的虚拟节点，然后完成了虚拟节点到真实节点的映射。假设现在计算得出了位置D，那么按照顺时针的顺序，我们找到的第一个节点就是**C #1**，最终数据实际还是会落在节点C上。

通过增加虚拟节点的方式，使ABC三个节点在圆环上的位置更加均匀，平均了落在每一个节点上的概率。这样一来就解决了上文提到的数据存储存在不均匀的问题了，这就是一致性哈希的虚拟节点机制。



### 类一致性哈希算法

上面提到，Redis-Cluster使用的是类哈希算法。至于为什么是类一致性哈希算法，是因为两者间有些许差别。

**一致性哈希是对2^32取模，而Redis Cluster则是对2^14（也就是16384）取模。**Redis Cluster将自己分成了16384个**Slot**（槽位）。通过CRC16算法计算出来的哈希值会跟16384取模，取模之后得到的值就是对应的槽位，**然后每个Redis节点都会负责处理一部分的槽位**，就像下表这样。

| 节点 | 处理槽位      |
| ---- | ------------- |
| A    | 0 - 5000      |
| B    | 5001 - 10000  |
| C    | 10001 - 16383 |

每个Redis实例会自己维护一份**slot - Redis节点**的映射关系，假设你在节点A上设置了某个key，但是这个key通过CRC16计算出来的槽位是由节点B维护的，那么就会提示你需要去节点B上进行操作。

![image-20210203230855063](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210203230855063.png)



# 集群配置与搭建

Redis-Cluster一般由多个节点组成，节点数量至少是6个（3个master，3个slave）才能保证组成完整高可用的集群。每个节点需要开启配置`cluster-enable yes`，该配置在 **redis.conf**可见。



其他配置和单机模式一致即可，配置文件命名规则 `redis-{port}.conf`，准备好配置后启动所有节点，第一次启动时如果没有集群配置文件，它会自动创建一份，文件名称采用 `cluster-config-file` 参数项控制，建议采用 `node-{port}.conf` 格式定义，也就是说会有两份配置文件

当集群内节点信息发生变化，如添加节点、节点下线、故障转移等。**节点会自动保存集群状态到配置文件中**。需要注意的是，Redis 自动维护集群配置文件，不要手动修改，防止节点重启时产生集群信息错乱

```conf
# 节点端口 
port 6379 

# 开启集群模式
cluster-enabled yes

# 节点超时时间，单位毫秒
cluster-node-timeout 15000 

# 集群内部配置文件
cluster-config-file "nodes-6379.conf"
```

集群配置文件的内容，大概如下图所示

![image-20210204144747267](C:%5CUsers%5C99380%5CAppData%5CRoaming%5CTypora%5Ctypora-user-images%5Cimage-20210204144747267.png)

文件记录了集群初始状等信息。最为重要的节点ID，它是一个长度为40的16进制字符串（没错这个就是Redis实例的运行ID），用于唯一标识集群内的一个节点。节点ID在集群初始化时只创建一次，节点重启时会加载集群配置文件进行重用，结合做相应的集群操作。



搭建redis-cluster主要有几个步骤

- 配置文件
- 节点握手
- 设置从节点
- 分配slot槽



## 节点握手

redis-cluster并没有内置节点自动发现机制，而是采用了gossip协议（流言协议），来进行节点间的通信。有关这个gossip协议，下文再作介绍。



由客户端发起命令 `cluster meet{ip}{port}`进行节点握手，表示从当前连接的节点，与命令中的节点进行握手。

```shell
# 192.168.1.100:7000 与 192.168.1.100:7005 进行握手
[root@8gVm redis-3.0.4]# src/redis-cli -c -h 192.168.1.100 -p 7000 cluster meet 192.168.1.100 7005
```

- 节点7000与节点7005进行握手，并由7000节点发送meet信息
- 节点7005收到meet信息后，保存7000节点信息，并回复pong信息
- 之后节点7000与节点7005节点定期通过 `ping/pong`消息进行正常的节点通信。



最后使用 `cluster nodes`命令确认节点都彼此感知并组成集群。

![image-20210204163748831](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210204163748831.png)

注意：

- 若集群内有五个节点，只需要由一个节点发送4次 `cluster meet`命令即可。
- cluster meet命令是异步命令，执行后会立刻返回。由内部发起与目标节点进行握手
- 每个节点会占用两个TCP端口，一个监听客户端的请求，由redis.conf中的 `port配置项`指定，默认6379，另一个在前一个端口加上10000，比如16379，来监听数据的请求，节点与节点间会监听第二个端口，此采用gossip协议进行通信
- 节点间通过第二个端口来进行失败检测，配置更新，failover认证等操作
- 节点建立连接后，集群还不能正常工作，此时集群处于下线状态。所有的数据读写都被禁止。

## 设置从节点

在Redis-Cluster中，需要配置主从节点，保证当master故障时，自动进行故障转移。首次启动的节点和被分配槽的节点都是主节点，从节点负责复制主节点槽信息和相关数据。

**slave节点只是充当了一个数据备份的角色，当master发生了宕机，就会将对应的slave节点提拔为master，来重新对外提供服务。**



使用 `cluster replicate {nodeId}`命令，将当前节点设置为nodeId指定节点的slave节点。

一旦一个节点变成另一个master节点的slave，无需通知群集内其他节点这一变化：节点间交换 信息的心跳包会自动将新的配置信息分发至所有节点。

## 分配slot槽

Redis集群把所有数据映射到16384个槽中（2^14），每个key会映射为一个固定的槽，只有当节点分配了槽，才能响应和这些槽关联的键命令。

**通过`cluster addslots`命令为节点分配槽，可以利用bash特性批量设置**

```shell
redis-cli -h 127.0.0.1 -p 6379 cluster addslots {0..5461}
redis-cli -h 127.0.0.1 -p 6380 cluster addslots {5462..10922}
redis-cli -h 127.0.0.1 -p 6381 cluster addslots {10923..16383}
```

把16384个slot平均分配给三个节点，执行cluster info命令查看集群状态

![image-20210204175123050](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210204175123050.png)

如上图所示，集群状态是ok的，集群进入在线状态。

所有的槽都已经分配给节点，执行 `cluster nodes`命令可以看到节点和槽的分配关系

![image-20210204181004687](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210204181004687.png)



## cluster命令总结

**集群**

- cluster info ：打印集群的信息

- cluster nodes ：列出集群当前已知的所有节点（ node），以及这些节点的相关信息。

**节点**

- cluster meet <ip> <port> ：将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。
- cluster forget <node_id> ：从集群中移除 node_id 指定的节点。
- cluster replicate <master_node_id> ：将当前从节点设置为 node_id 指定的master节点的slave节点。只能针对slave节点操作。
- cluster saveconfig ：将节点的配置文件保存到硬盘里面。

**槽(slot)**

- cluster addslots <slot> [slot ...] ：将一个或多个槽（ slot）指派（ assign）给当前节点。
- cluster delslots <slot> [slot ...] ：移除一个或多个槽对当前节点的指派。
- cluster flushslots ：移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。
- cluster setslot <slot> node <node_id> ：将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给
  **另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。**
- cluster setslot <slot> migrating <node_id> ：将本节点的槽 slot 迁移到 node_id 指定的节点中。
- cluster setslot <slot> importing <node_id> ：从 node_id 指定的节点中导入槽 slot 到本节点。

- cluster setslot <slot> stable ：取消对槽 slot 的导入（ import）或者迁移（ migrate）。

**键**

- cluster keyslot <key> ：计算键 key 应该被放置在哪个槽上。
- cluster countkeysinslot <slot> ：返回槽 slot 目前包含的键值对数量。
- cluster getkeysinslot <slot> <count> ：返回 count 个 slot 槽中的键 。



## 使用客户端工具搭建集群

https://blog.csdn.net/qq_41453285/article/details/106451296

上面搭建集群的命令和步骤繁多，当节点过多时，必然会加大搭建的复杂度和运维成本。因此redis提供了`redis-cli --cluster`来搭建

ps：`redis-cli --cluster `命令本来是由redis-trib.rb工具提供的，但是随着发展，redis-trib.rb工具的功能逐渐被归纳到redis-cli工具中。因此本文不再介绍redis-trib.rb工具。

### 1. 准备节点

- 与上面一样，先关闭上面的所有节点，重新启动6个集群节点进行实验

```shell
sudo redis-server /opt/redis/conf/redis-6379.conf
sudo redis-server /opt/redis/conf/redis-6380.conf
sudo redis-server /opt/redis/conf/redis-6381.conf
sudo redis-server /opt/redis/conf/redis-6382.conf
sudo redis-server /opt/redis/conf/redis-6383.conf
sudo redis-server /opt/redis/conf/redis-6384.conf
```

![image-20210204182833146](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210204182833146.png)



### 2. 创建集群

**输入下面的命令自动完成节点握手和槽分配。命令如下：**

- --cluster-replicas 1：指定集群中每个主节点配备几个从节点，这里设置为1。并且该命令会自己创建主节点和分配从节点，其中前3个是主节点，后3个是从节点，后3个从节点分别复制前3个主节点

```shell
# 创建3个集群主节点和3个集群从节点
redis-cli --cluster create --cluster-replicas 1 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384
```

输入上述命令后，中间会让你输入 yes or no，下面输入yes开始执行节点握手和槽分配

![image-20210204183100323](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210204183100323.png)

- 最后的输出报告说明：16384个槽全部被分配，集群创建成功。这里需要注意命令中节点的地址必须是不包含任何槽/数据的节点，否则会拒绝创建集群
- **备注：**如果只想创建主节点，而不同时创建从节点，那么需要忽略--cluster-replicas 1参数。命令如下，例如下面只创建3个集群主节点

```shell
# 只创建3个集群主节点
redis-cli --cluster create 127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381
```

### 集群完整性检查

- 集群完整性指所有的槽都分配到存活的主节点上，**只要16384个槽中有 一个没有分配给节点则表示集群不完整**
- 可以使用下面的命令检测之前创建的两个集群是否成功，check命令只需要给出集群中任意一个节点地址就可以完成整个集群的检查工作：

```shell
redis-cli --cluster check 127.0.0.1:6379
```

![image-20210204183243902](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210204183243902.png)

上图表示所有slot都已分配到节点



## 配置参数详解

- \- cluster-enabled `<yes/no>`: 是否启用集群模式，默认是注释掉的，相当于no使用单机模式
- cluster-config-file `<filename>`: 集群配置文件名，集群会自动生成和更新这个文件，不建议手动修改，用于记录集群的基本信息，例如状态，持久化变量等等。注意不要和其它配置文件同名，一般使用nodes-6379.conf，6379是使用的端口
- cluster-node-timeout `<milliseconds>`: 集群节点的不可用时间(超时时间)，超过这个时间就会被认为下线，单位是毫秒，一般是15000毫秒
- cluster-slave-validity-factor `<factor>`: 集群主从切换的控制因素之一，需要联合上述cluster-node-timeout参数一起使用。如果设为0，当主节点下线的时候，不管从节点与主节点断开连接的时间有多久，集群都会尝试主从切换。如果该值设为大于0，主从间的最大断线时间会通过node timeout x cluster-node-timeout计算。例如，如果cluster-node-timeout=5000，cluster-slave-validity-factor=10，那么如果一个从节点与主节点断线超过5000x10=50000ms=50s的话，当主节点下线的时候，该从节点都不会被切换为主节点。需要注意的是，如果该值设为非0的话，在一个主节点下线的时候，集群有可能会因为从节点不能进行切换导致不能正常运作。在这种情况下，集群只能在原主节点重新上线连接到集群的时候才能恢复正常运作
- cluster-migration-barrier` <count>`: 与同一主节点连接的从节点最少个数，如果实际连接到同一个主节点的从节点个数超过该值，多余的从节点可以被迁移到其它没有从节点连接的主节点
- cluster-require-full-coverage` <yes/no>`: 默认值为yes，意思是需要集群内的全部hash slots都正常工作才能接收写命令，如果设为0，那么可以允许部分hash slots下线的情况下继续执行读操作

## 新节点加入集群

在集群运行过程中，可以通过两种方式新增节点

- cluster meet命令，新节点与集群中的节点进行握手，然后通过集群节点间的ping命令传播
- redis-cli --cluster命令，该命令的本质也是cluster meet

```shell
redis-cli --cluster add-node new_host:new_port existing_host:existing_port --cluster-slave --cluster-master-id <arg>
 
# 例如下面将6385加入到6379所属的集群中，并且作为117457eab5071954faab5e81c3170600d5192270的从节点
redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379 --cluster-slave --cluster-master-id 117457eab5071954faab5e81c3170600d5192270
```

- --cluster-slave和--cluster-master-id是可选的，在设置从节点的时候才会用。如果不指定--cluster-master-id会随机分配到任意一个主节点



**正式环境建议使用redis-cli cluster命令加入新节点**，该命令内部会执行新节点状态检查，如果新节点已经加入其他集群或者包含数据，则放弃集群加入操作并打印如下信息：

![image-20210207104340419](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207104340419.png)



如果我们手动执行cluster meet命令加入已经存在于其他集群的节点，会造成被加入节点的集群合并到现有集群的情况，从而造成数据丢失和错乱， 后果非常严重，线上谨慎操作



## 集群的功能限制

Cluster在功能上相对单机来说存在一些限制，在此处稍作总结

- **1）key批量操作支持有限。**如mset、mget，目前只支持具有相同slot值的 key执行批量操作。对于映射为不同slot值的key由于执行mget、mget等操作可 能存在于多个节点上因此不被支持
- **2）key事务操作支持有限。**同理只支持多key在同一节点上的事务操 作，当多个key分布在不同的节点上时无法使用事务功能
- 3）key作为数据分区的最小粒度，因此**不能将一个大的键值对象如 hash、list等映射到不同的节点**
- **4）不支持多数据库空间**。单机下的Redis可以支持16个数据库，集群模 式下只能使用一个数据库空间，即db0
- **5）复制结构只支持一层，**从节点只能复制主节点，不支持嵌套树状复 制结构



## 总结

本文着重介绍如何redis-cluster结构和如何搭建一个redis-cluster，至于cluster内部原理，则放在一篇再进行详细介绍。

搭建步骤：

- 准备至少6个节点，开启cluster-enable配置项
- 节点握手, `cluster meet{ip}{port}`

- 分配slot槽， **通过`cluster addslots`命令为节点分配槽，可以利用bash特性批量设置**``

- 集群完备性检测， `cluster info` 和 `cluster nodes`



## 参考文献

https://blog.csdn.net/weixin_42667608/article/details/111360617

https://mp.weixin.qq.com/s/Z-PyNgiqYrm0ZYg0r6MVeQ

https://blog.csdn.net/hguisu/article/details/82979320

https://blog.csdn.net/qq_41453285/article/details/103402727