##

集群之槽信息的记录（slots属性、numslot属性、CLUSTER ADDSLOTS命令的实现）

https://dongshao.blog.csdn.net/article/details/103365159



集群之计算键所属槽(CLUSTER KEYLOST命令)、判断槽所属节点、MOVED错误

https://blog.csdn.net/qq_41453285/article/details/103394429



集群之节点数据库的实现（slots_to_keys跳跃表、CLUSTER GETKEYSINSLOT命令）

https://dongshao.blog.csdn.net/article/details/103394723



## 概述

在进行集群拓容的时候，共有下面几个步骤：

- 准备需要加入的节点
- 节点加入到集群（两种方式）
- 准备槽迁移
  - 从集群中已有的节点提取slot
  - 找到所有属于这些slot的key
  - 将这些key一一移动到新节点
  - 修改slot从属关系
  - 在集群中广播新节点slot

- 新节点上线

ps：节点下线，集群缩容操作中，操作步骤与集群拓容相反，详细看《Redis-Cluster搭建-集群伸缩》,这里点到即止。



下面就要深入其中的原理

## 节点加入

有两种方式

- cluster meet <ip> <port> 命令
- redis-cli --cluster 工具，其本质也是cluster meet

详情看《Redis-Cluster原理-节点握手》，这里点到即止

---

随后就是从已有的节点中分配slot到新的节点，那么节点与slot的关系又保存到哪里？

在集群中，每个Redis节点有维护有 `clusterNode、clusterState`节点，其中几个重要属性，用于维护节点和slot的关系。

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

在 `clusterNode`结构中，slot属性和numslots属性，记录了节点负责处理哪些槽

## clusterNode-slot属性与numSlots属性

- slots属性是一个**二进制位数组（bit array）**，这个数组的长度为16384/8=2048个字节，**共包含16384个二进制位**
- Redis以0为起始索引，16383为终止索引，对slots数组中的16384个二进制位进行编号，并根据索引i的二进制位的值来判断是否负责处理槽i。
  - 当为1时， 表明节点负责处理槽i
  - 当为0时，表明节点不处理槽i

如图，这个数组索引0至索引7上的二进制位的值都为1，其余所有二进制位的值都为0，**这表示节点负责处理槽0至槽7**

![image-20210208150311339](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208150311339.png)



因为取出和设置slots数组中的**任意一个二进制位的值的复杂度仅为O（1）**，所以对于一 个给定节点的slots数组来说，程序检查节点是否负责处理某个槽，又或者将某个槽指派给节 点负责，这**两个动作的复杂度都是O（1）**



至于numslots属性则记录节点**负责处理的槽的数量**，也**即是slots数组中值为1的二进制位的数量**，通过这个属性可以以 O(1)复杂度获取到当前节点负责的slot，无需遍历slot属性

## 指定槽迁移

上述两个属性，可以查看当前节点所负责的槽。新节点加入集群后，使用两个命令指定槽的迁移

- `CLUSTER SETSLOT <slot> IMPORTING <source_id>` ，让目标节点准备好从源节点导入槽slot的键值对
- `CLUSTER SETSLOT <slot> MIGRATING <target_id>` ，让源节点准备好将属于槽slot的键值对迁移到目标节点。

### cluster setslot <slot> IMPORTING <source_id>

clusterState结构的 **importing_slots_from** 数组记录了当前节点正从其他节点导入的槽

- 如果migrating_slots_from[i]的**值不为NULL，而是指向一个clusterNode结构**，那么表示当前节点正在从clusterNode所代表的节点导入槽

- 在对集群进行重新分片的时候，向目标节点发送命令，可以**将目标节点clusterState.importing_slots_from[i]的值设置为source_id所代表节点的clusterNode结构**

### CLUSTER SETSLOT <slot> MIGRATING <target_id>

clusterState结构的 **migrating_slots_to **数组记录了当前节点正在迁移至其他节点的槽

- 如果migrating_slots_to[i]的**值不为NULL，而是指向一个clusterNode结构**，那么表示当前 节点正在将槽i迁移至clusterNode所代表的节点

- 在对集群进行重新分片的时候，向源节点发送命令，可以**将源节点clusterState.migrating_slots_to[i]的值设置为target_id所代表节点的 clusterNode结构**

  

## slots_to_keys跳跃表

**节点中使用clusterState结构中的slots_to_keys跳跃表来保存槽和键之间的关系。**

```c
typedef struct clusterState {
    // ...
    zskiplist *slots_to_keys;
    // ...
} clusterState;
```

有关跳表 zskipList，请看《Redis底层数据结构》篇，这里点到即止

slots_to_key s跳跃表**每个节点的分值（score）都是一个槽号，而每个节点的成员 （member）都是一个数据库键：**

- 每当节点往数据库中添加一个新的键值对时，节点就会将这个键以及键的槽号关联到 slots_to_key s跳跃表
- 当节点删除数据库中的某个键值对时，节点就会在slots_to_keys跳跃表解除被删除键与 槽号的关联

![image-20210208160611277](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208160611277.png)

### CLUSTER GETKEYSINSLOT命令

通过slots_to_keys跳跃表中记录各个数据库键所属的槽，节点可以很方便对属于某个或某些槽的所有key进行批量操作。

`CLUSTER GETKEYSINSLOT <slot> <count>`命令**可以返回最多count个属于槽slot的数据库键**，而**这个命令就是通过遍历 slots_to_keys跳跃表来实现的**

在RedisCluster槽迁移的时候，就是利用 CLUSTER GETKEYSINSLOT 命令，来获取到需要迁移的槽里面的所有key

当拿到这些key后，就可以进行key的迁移了。



## 键迁移

通过`CLUSTER GETKEYSINSLOT <slot> <count>`获取到迁移槽里面所有的key，就可以用命令 `MIGRATE <target_ip> <target_port> <key_name> 0 <time out>`进行单个key的迁移



![image-20210208173852975](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210208173852975.png)

注意，key是逐个进行迁移的