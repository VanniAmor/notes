[toc]



# **dict-字典**

Redis的一个database中，所有的**Key到Value的映射，就是使用一个dict来维护的**。字典的本质就是为了解决算法中的查找问题。

```html
#1.一般查找问题的解法分为两个大类
    一个是基于各种平衡树，一个是基于哈希表，平常使用的各种Map或dictionary，大都是基于哈希表实现的。
 
#2.dict的算法实现
    dict也是一个基于哈希表的算法，跟java中的hashMap类似，dict采用某个哈希函数从key计算得到在哈希表中的位置，采用拉链法解决冲突，
并在装载因子（load factor）超过预定值时自动扩展内存，引发重哈希（rehashing）。
```

【博客参考】https://blog.csdn.net/Xiejingfa/article/details/51018337?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-4.control&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-4.control

---

Redis定义了dictEntry、dictType、dictht和dict四个结构体来实现散列表的功能。

（1）dictEntry结构体

```c
/* 保存键值（key - value）对的结构体，类似于STL的pair。*/
typedef struct dictEntry {
    // 关键字key定义
    void *key;  
    // 值value定义，只能存放一个被选中的成员
    union {
        void *val;      
        uint64_t u64;   
        int64_t s64;    
        double d;       
    } v;
    // 指向下一个键值对节点
    struct dictEntry *next;
} dictEntry;
```

从dictEntry的定义我们也可以看出**dict通过“拉链法”来解决冲突**问题。

所谓“拉链法”， 即链表与数组结合，即创建一个链表数组，数组中每个元素就是一个链表，当key遇到hash冲突时，则将冲突的值放到链表中

![image-20210116123322641](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210116123322641.png)

（2） dictType结构体

```c
/* 定义了字典操作的公共方法，类似于adlist.h文件中list的定义，将对节点的公共操作方法统一定义。搞不明白为什么要命名为dictType */
typedef struct dictType {
    /* hash方法，根据关键字计算哈希值 */
    unsigned int (*hashFunction)(const void *key);
    /* 复制key */
    void *(*keyDup)(void *privdata, const void *key);
    /* 复制value */
    void *(*valDup)(void *privdata, const void *obj);
    /* 关键字比较方法 */
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    /* 销毁key */
    void (*keyDestructor)(void *privdata, void *key);
    /* 销毁value */
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

（3） dictht结构体

```c
/* 哈希表结构 */
typedef struct dictht {
    // 散列数组。
    dictEntry **table;
    // 散列数组的长度
    unsigned long size;
    // sizemask等于size减1
    unsigned long sizemask;
    // 散列数组中已经被使用的节点数量
    unsigned long used;
} dictht;
```

（4） dict结构体

```c
/* 字典的主操作类，对dictht结构再次包装  */
typedef struct dict {
    // 字典类型
    dictType *type;
    // 私有数据
    void *privdata;
    // 一个字典中有两个哈希表
    dictht ht[2];
    // 数据动态迁移的下标位置
    long rehashidx; 
    // 当前正在使用的迭代器的数量
    int iterators; 
} dict;
```



各个结构体的关系如下

![image-20210116130238124](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210116130238124.png)

## 字典的插入

- 先用哈希函数计算键key的hash值（Redis采用的是MurMurHash2算法来生成哈希值）

- 借助sizemask和哈希值，计算出索引值

  index = hash & ht[x].sizemask

- 上面计算出来的index值其实就是对应dictEntry* 数组的下标，如果对应下标没有存放任何键值对，则直接存放，否则借助开链法，从链表头插入新的键值对（因为链表没有记录指向链表尾部的指针，所以从链表头插入的效率更高，可以达到O(1)）

## **字典的Rehash**

当哈希表的冲突率过高时，链表会很长，这时查询效率会变得很低，就有必要进行哈希表的拓张。而如果哈希表存放的键很少，但是size设得很大，又会浪费空间，这时也有必要进行哈希表收缩。这里的拓张和收缩，就是rehash

简单概括rehash的过程，可以总结为下面几点：

1. 为dict的哈希表ht[1]分配空间，分配的空间大小取决于操作类型和当前键值对数量ht[0].used

   (1)如果是扩展操作，ht[1]的大小为**第一个大于**等于![ht[0].used*2*2^{n}](https://private.codecogs.com/gif.latex?ht%5B0%5D.used*2*2%5E%7Bn%7D)的整数

   (2)如果是收缩操作，ht[1]的大小为**第一个大于**等于![ht[0].used*2^{n}](https://private.codecogs.com/gif.latex?ht%5B0%5D.used*2%5E%7Bn%7D)的整数

2. 重新计算ht[0]中的所有键的哈希值和索引值，将相应的键值迁移到ht[1]指定位置中去，需要注意的是，这个过程是**渐进式的**，不然如果字典很大的话，全部迁移需要很长时间，这段时间内Redis无法提供服务。

3. 当ht[0]的所有键值对都迁移到ht[1]中去后（此时ht[0]会变成空表），把ht[1]设置为ht[0]，并重新在ht[1]上新建一个空表，为下次rehash做准备



【博客】 https://www.cnblogs.com/jaycekon/p/6227442.html

```
渐进rehash的具体过程如下：
1. 为ht[1]分配空间，此时字典同时持有ht[0]和ht[1]
2. 将rehashidx设为0，标志rehash正式开始
3. 在rehash期间，每次对字典执行任意操作时，程序除了执行对应操作之外，还会顺带将ht[0]在rehashidx索引上的所有键值对rehash到ht[1]，操作完后rehashidx + 1
4. 在rehash期间，对字典进行ht[0].size次操作之后，rehashidx的值会增加到ht[0].size，此时ht[0]的所有键值对都已经迁移到ht[1]了，程序会将rehashidx重新置为-1，以此表示rehash完成

```

需要注意的是，在rehash过程中，ht[0]和ht[1]可能同时存在键值对，因此在执行查询操作过程中需要同时查询两个哈希表，而在执行插入操作时， 则直接在ht[1]上操作即可。

####  **触发rehash的条件**

- 服务器当前**没有在执行BGSAVE**或**BGREWRITEAOF**命令且哈希表的**负载因子大于等于1**时进行扩展操作
- 服务器**正在执行BGSAVE或BGREWRITEAOF命令**且哈希表**的负载因子大于等于5**时进行扩展操作
- 当前**负载因子小于0.1**时进行收缩操作

负载因子 = 哈希表当前保存节点数 / 哈希表大小（长度）

之所以在执行BGSAVE或BGREWRITEAOF时，负载因子较大，是因为此时Redis会创建子进程，而大多数操作系统采取了写时复制的技术来优化子进程使用效率，不适合在此时做大规模数据迁移活动，简单来说就是节约内存和提升效率



https://blog.csdn.net/cxc576502021/article/details/82974940?utm_medium=distribute.wap_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.wap_blog_relevant_pic&depth_1-utm_source=distribute.wap_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-4.wap_blog_relevant_pic



## 参考文献

https://blog.csdn.net/seky_fei/article/details/106968173



