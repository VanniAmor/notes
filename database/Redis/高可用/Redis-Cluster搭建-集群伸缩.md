https://dongshao.blog.csdn.net/category_9536747_2.html



https://dongshao.blog.csdn.net/category_9987850_2.html



## 概述

Redis-Cluster提供了灵活的节点拓容和收缩方案。在不影响集群对外服务的情况下，可以为集群添加节点进行拓容也可以下线部分节点进行缩容。

![image-20210207104557259](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207104557259.png)

而这种不影响集群服务的操作，就是动态伸缩。

集群拓容是分布式存储中最为常见的需求，Redis集群拓容操作可以简单分为如下步骤

- 准备新节点
- 新节点加入集群
- 迁移槽和数据



## 节点新增

假设有如下集群和节点，现吧6385和6386加入集群，6385为master，6386为slave

ps：新增节点前，务必确保新节点是空的，且没有加入到其他集群中

![image-20210207113141542](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207113141542.png)

使用 cluster meet命令加入到集群中。

```shell
redis-cli -p 6379 cluster meet 127.0.0.1 6385
redis-cli -p 6379 cluster meet 127.0.0.1 6386
```

使用redis-cli --cluster命令加入到集群

```c
redis-cli --cluster add-node new_host:new_port existing_host:existing_port --cluster-slave --cluster-master-id <arg>
 
# 例如下面将6385加入到6379所属的集群中，并且作为117457eab5071954faab5e81c3170600d5192270的从节点
redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379 --cluster-slave --cluster-master-id 117457eab5071954faab5e81c3170600d5192270
```

或者

```c
# 将6385添加到6379所在的集群中
redis-cli --cluster add-node 127.0.0.1:6385 127.0.0.1:6379
 
# 将6386添加到6379所在的集群中
redis-cli --cluster add-node 127.0.0.1:6386 127.0.0.1:6379
```

新增节点完成后，由于没有给该节点分配槽，所以该节点还是无法提供读写服务的。

所以接下来要进行槽的迁移



### 节点槽迁移（集群重新分片）

Redis集群的重新分片操作可以将任意数量已经指派给某个节点的槽，并且相关槽所属的键值对也会从源节点被移动到目标节点中。

重新分片操作可以在线进行，在重新分片过程中，集群不需要下线，并且源节点和目标节点可以继续处理命令请求。



因为迁移是从现有的节点中分配出slot给新节点，所以需要合理设计每个节点slot的数量，**尽量保证各个节点的数据是均匀的。**



如下图，新增节点是6385，需要从现有的三个节点6379,6380,6381中分配槽给新节点。

![image-20210207171552560](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207171552560.png)

有两种方式进行槽迁移操作

- 原生命令
- 使用redis-cli --cluster工具

数据的迁移过程是逐个槽进行的。



### 原生命令

- 1）对目标节点发送cluster setslot {slot} importing {sourceNodeId}命令，让 目标节点准备导入槽的数据
- 2）对源节点发送cluster setslot {slot} migrating {targetNodeId}命令，让源 节点准备迁出槽的数据
- 3）源节点循环执行cluster getkeysinslot {slot} {count}命令，获取count个属于槽{slot}的键
- 4）在源节点上执行migrate {targetIp} {targetPort} "" 0 {timeout} keys {keys...}命令，把获取的键通过流水线（pipeline）机制批量迁移到目标节点，批量 迁移版本的migrate命令在Redis3.0.6以上版本提供，之前的migrate命令只能 单个键迁移。对于大量key的场景，批量键迁移将极大降低节点之间网络IO次数
- 5）重复执行步骤3）和步骤4）直到槽下所有的键值数据迁移到目标节点
- 6）向集群内所有主节点发送cluster setslot {slot} node {targetNodeId}命令，通知槽分配给目标节点。为了保证槽节点映射变更及时传播，需要遍历发送给所有主节点更新被迁移的槽指向新节点

![image-20210207173633480](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207173633480.png)

这种直接用命令的操作方式极其繁琐，特别是当slot上面key的数量很多，而且集群中节点数量多的时候，很容易会出现纰漏

所以redis提供了一个工具，来进行槽迁移工作，即redis-trib.rb，这个是基于Ruby的。现在这个工具已经并入了Redis的客户端工具。可以直接使用redis-cli --cluster来使用相关功能。



### redis-cli工具

相关命令如下

- host：port：必传参数，集群内任意节点地址，用来获取整个集群信息
- --from：制定源节点的id，如果有多个源节点，使用逗号分隔，如果是all源节点变为集群内所有主节点，在迁移过程中提示用户输入
- --to：需要迁移的目标节点的id，目标节点只能填写一个，在迁移过程 中提示用户输入
- --slots：需要迁移槽的总数量，在迁移过程中提示用户输入
- --yes：当打印出reshard执行计划时，是否需要用户输入yes确认后再执行reshard
- --timeout：控制每次migrate操作的超时时间，默认为60000毫秒
- ·--pipeline：控制每次批量迁移键的数量，默认为10

```shell
redis-cli --cluster reshard host:port --from <arg> --to <arg> --slots <arg> --yes --timeout
<arg> --pipeline <arg>
```

迁移完成之后，rehash会自动退出。

槽用于hash运算，本身顺序没有意义，因此无须强制要求节点负责槽的顺序性。迁移之后建议使用下面的命令检查节点之间槽的均衡性。

```shell
redis-cli --cluster rebalance 127.0.0.1:6380
```

## 集群收缩

![image-20210207181232395](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207181232395.png)

- 1）首先需要确定下线节点是否有负责的槽，如果是，需要把槽迁移到 其他节点，保证节点下线后整个集群槽节点映射的完整性
- 2）当下线节点不再负责槽或者本身是从节点时，就可以通知集群内其 他节点忘记下线节点，当所有的节点忘记该节点后可以正常关闭

假设集群中节点关系如下

![image-20210207181555953](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207181555953.png)

**现在我们想把6381和6382节点进行下线**（其中6381是主节点，6382复制6381）

### 下线槽迁移

以要下线的节点为源节点，其他节点为目标节点，进行槽迁移工作。

使用rehash命令完成槽迁移，由于要将6381的槽均匀分配到其他三个节点中，因此需要输入3次rehash命令，每次迁移的目标节点ID（redis实例运行ID）不同

![image-20210207182641942](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207182641942.png)



具体操作看上面的槽迁移

### 忘记节点

由于集群内的节点不停地通过Gossip消息彼此交换节点状态，**因此需要通过一种健壮的机制让集群内所有节点忘记下线的节点。**也就是说让其他节点不再与要下线节点进行Gossip消息交换

Redis提供了clusterforget {downNodeId} 命令实现该功能

![image-20210207183012603](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210207183012603.png)

- 当节点接收到cluster forget {down NodeId}命令后，会把nodeId指定的 点加入到禁用列表中，在禁用列表内的节点不再发送Gossip消息。禁用列表有效期是60秒，超过60秒节点会再次参与消息交换。也就是说当第一次forget命令发出后，我们有60秒的时间让集群内的所有节点忘记下线节点
- **线上操作不建议直接使用cluster forget命令下线节点，需要跟大量节点命令交互，**实际操作起来过于繁琐并且容易遗漏forget节点
- **建议使用redis-cli --cluster del-node {host：port} {downNodeId}命令，**内部实现的伪代码如下：

```c
def delnode_cluster_cmd(downNode):
    # 下线节点不允许包含slots
    if downNode.slots.length != 0
        exit 1
    end
    # 向集群内节点发送cluster forget
    for n in nodes:
        if n.id == downNode.id:
            # 不能对自己做forget操作
            continue;
        # 如果下线节点有从节点则把从节点指向其他主节点
        if n.replicate && n.replicate.nodeId == downNode.id :
            # 指向拥有最少从节点的主节点
            master = get_master_with_least_replicas();
            n.cluster("replicate",master.nodeId);
        #发送忘记节点命令
        n.cluster('forget',downNode.id)
    # 节点关闭
    downNode.shutdown();
```

从伪代码看出del-node命令帮我们实现了安全下线的后续操作。当下线主节点具有从节点时需要把该从节点指向到其他主节点，**因此对于主从节点都下线的情况，建议先下线从节点再下线主节点，防止不必要的全量复制。**

## MOVED错误

当节点发现键所在的槽**并非由自己负责处理的时候**，节点就会向客户端返回一个 MOVED错误，指引客户端转向至正在负责槽的节点

格式为：`MOVED <slot> <ip>:<port>`

- 其中slot为**键所在的槽**，而ip和port则是**负责处理槽**slot的节点的IP地址和端口号

![image-20210208094114683](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208094114683.png)

- 当客户端接收到节点返回的MOVED错误时，客户端会根据MOVED错误中提供的IP和端口号，转向至负责处理slot的节点，并向该节点重新发送之前要执行的命令
- 一个集群中，客户端通常会与多个节点创建套接字链接，而所谓的节点转向实际上就是换一个套接字来发送命令
- 如果客户端尚未和想要转向的节点创建套接字链接，那么会根据MOVED错误中的IP和端口来连接节点，然后再进行转向。

### 客户端不同运行模式下的错误显示

- 集群模式的redis-cli在接收到moved错误时，不会打印出MOVED错误，而是根据MOVED错误自动进行节点转向，并打印出转向信息。

![image-20210208101824802](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208101824802.png)

- 如果使用单机模式的redis-cli客户端，MOVED错误会被打印出来，且不会自动进行转向。因为单机模式redis-cli不清楚MOVED错误的作用，所以它只会将其打印出来。

  ![image-20210208101953136](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208101953136.png)

## ACK错误

在进行重新槽的重新分配时，比如A的槽迁移到B时，可能会出现一种情况，A的部分键值仍在A节点上，而部分节点已经迁移到B节点。

此时当客户端访问某个Key，而这个Key正处于A的slot下，有两种情况

- 这个Key仍在A节点，A节点直接执行命令
- 在A节点找不到Key，表明这个键可能已经迁移到目标节点，向客户端返回一个ASK错误，指引客户端转向正在导入槽的目标节点。并再次发生命令

![image-20210208112651409](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208112651409.png)

### 客户端不同运行模式下的错误显示

- 集群模式的redis-cli在接收到ASK错误时也不会打印，而是自动根据错误提供的IP和端口进行转向动作

- 单机模式下的redis-cli会打印ASK错误

  

## ASKING命令（REDIS_ASKING标识）

当客户端接收到ASK错误并转向正在导入槽的节点时，客户端会先向节点发送一个ASKING命令，然后再重新发送想要执行的命令。

![image-20210208125538025](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208125538025.png)



ASKING命令，会打开发送该客户端的REDIS_ASKING标识。



**因为如果客户端不发送ASKING命令，**而直接发送想要执行的命令的话，那么客户端发送的命令就会被拒绝，并返回一个MOVED错误。

且客户端的REDIS_ASKING标识是一个一次性标识，当**节点执行了一个带有REDIS_ASKING标识的客户端发送的命令之后，客户端的REDIS_ASKING标识就会被移除。**