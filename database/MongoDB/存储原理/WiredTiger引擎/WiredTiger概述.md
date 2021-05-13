## 引擎概述



从MongoDB3.2开始，采用WiredTiger作为默认的存储引擎，其有以下优秀的特点

### B-Tree、page的数据格式

WiredTiger 采用了b-tree来组织管理数据， 一个集合的Namespace， 来关联到该集合的索引， 通过索引可以有效地将感兴趣的部分数据加载到内存中， 通常会放进Cache里面， 以备后续使用。 将数据从磁盘读入内存或者从内存flush到磁盘的基本操作单位是一个内存页。

### Transaction

Mongodb 对于事物的支持是逐渐迭代的

- **3.2版本它支持了单个collect的单文档的事务性**
- **3.6版支持了多文档的事务性**
- **4.0版本我们会得到跨collect的事务性。** 

```shell
begin_transaction；
    insert/update/delete/query
tranaction_commit or transaction_rollback
```

事物的语义可以保证在事物开始到结束的过程中的一系列操作， 要么成功， 要么回滚到第一条操作之前， 这对于数据的一致性是非常有用的。



### journal日志

**持久化数据的关键**

是引擎用于辅助存储的一种机制，它通常和checkpoint配合使用，在系统发生异常的时候，尽可能确保用户的数据不丢失

在dbpath目录下面的journal目录用来存放journal文件，该文件用来记录write-ahead redo日志。该目录下还包含一个用来保存最近队列数的文件。一次正常的shutdown会删除journal目录下的所有文件，而非正常的shutdown（比如崩溃）则不会删除文件。当mongod进程重启时，这些文件用来自动恢复数据库保证数据的一致性。 

当MongoDB刷新journal文件的写操作到数据文件时，会记录哪些journal写操作已经被刷新过。一旦journal文件中只包含被刷新过的写操作时，这个文件就不会再起到恢复数据的作用，MongoDB会删除它，或者将其回收用作新的journal文件



### block manager

block manager是Cache和磁盘相互交互的一个模块， 主要负责打开、关闭文件（包含索引文件、数据文件以及元数据等）以及page的读写等。



### cache

数据库的数据通常是很大的， 而可用的内存容量是有限的， 通常需要吧重要的数据以及读取过的数据放进内存里面， 以备后续使用。 随着系统的使用， 越来越多的内存页被放进了内存， 内存容量吃紧的时候， 需要将一部分数据移出内存。 
WiredTiger的Cache采用b-tree的方式组织，每个b-tree节点为一个page，root page是b-tree的根节点，internal page是b-tree的中间索引节点，leaf page是真正存储数据的叶子节点；**b-tree的数据以磁盘页为单位按需从磁盘加载或写入磁盘。**

### checkpoint

checkpoint是每隔一定的时间， 将集合的所有修改写入到磁盘， 在内存中， 将数据的修改写入到一个extent 里面， 多个extent组成一个page， 一个或者多个page构成了一个checkpoint。 这里要注意的是， 对于数据的update操作， 并不是直接在原来的数据页进行修改， 而是写入到新分配的page。






按照Mongodb默认的配置，￼WiredTiger的写操作会先写入Cache，并持久化到WAL(Write ahead log)，每60s或log文件达到2GB时会做一次Checkpoint，将当前的数据持久化，产生一个新的快照。Wiredtiger连接初始化时，首先将数据恢复至最新的快照状态，然后根据WAL恢复数据，以保证存储可靠性。

![image-20210420210353422](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20210420210353422.png)



