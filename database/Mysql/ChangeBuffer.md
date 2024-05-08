（1）MySQL数据存储包含内存与磁盘**两个部分**；

（2）内存缓冲池(buffer pool)以页为单位，缓存最热的数据页(data page)与索引页(index page)；

（3）InnoDB以变种LRU算法管理缓冲池，并能够解决“**预读失效**”与“**缓冲池污染**”的问题；



对于读请求，Mysql的Buffer Pool可以有效减少磁盘IO，那么针对写请求，就设计了【ChangeBuffer】这个技术，需要注意的是，ChangeBuffer也是在Buffer Pool中的



**redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的是随机读磁盘的IO消耗。**



## 概述

在MySQL5.5之前，叫插入缓冲(insert buffer)，只针对insert做了优化；现在对delete和update也有效，叫做写缓冲(change buffer)。

 

它是一种应用在**非唯一普通索引页**(non-unique secondary index page)不在缓冲池中，对页进行了写操作，并不会立刻将磁盘页加载到缓冲池，而仅仅记录缓冲变更(buffer changes)，等未来数据被读取时，再将数据合并(merge)恢复到缓冲池中的技术。写缓冲的目的是降低写操作的磁盘IO，提升数据库性能。



### 为何针对的是非唯一普通索引

如果索引设置了唯一(unique)属性，在进行修改操作时，**InnoDB必须进行唯一性检查**。也就是说，索引页即使不在缓冲池，磁盘上的页读取无法避免(否则怎么校验是否唯一？)，此时就应该直接把相应的页放入缓冲池再进行修改，而不应该再整写缓冲这个幺蛾子。



![image-20210731124307413](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731124307413.png)



当需要操作的记录没有命中Buffer Pool时，最起码需要一次磁盘IO。而且这种磁盘的读取，还是随机读，效率非常低。



当加入ChangeBuffer后，如下

![image-20210731131152901](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731131152901.png)



### 一致性问题

加入ChangeBuffer后，保证数据一致性的流程如下

（1）数据库异常奔溃，能够从redo log中恢复数据；

（2）写缓冲不只是一个内存结构，它也会被定期刷盘到写缓冲系统表空间；

（3）数据读取时，有另外的流程，将数据合并到缓冲池；



## 数据合并

所谓数据合并，两种情况

- 是指磁盘中读取的数据与ChangeBuffer中的数据合并后，放入到BufferPool中
- 数据回写，把ChangeBuffer中的数据合并到磁盘中



![image-20210731132121097](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731132121097.png)



![image-20210731132750582](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731132750582.png)



## 适用场景

- 数据库中都是唯一索引，不适用
- 写入数据后，立刻读取，不适用



- 数据库中有大量非唯一索引
- 业务写多读少，或者不是立刻读取



## 参数配置



**参数**：innodb_change_buffer_max_size

**介绍**：配置写缓冲的大小，占整个缓冲池的比例，默认值是25%，最大值是50%。



**参数**：innodb_change_buffering

**介绍**：配置哪些写操作启用写缓冲，可以设置成all/none/inserts/deletes等。



![image-20210731133108616](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731133108616.png)

## 参考文献

https://www.cnblogs.com/geaozhang/p/7235953.html

https://zhuanlan.zhihu.com/p/158879979

https://blog.csdn.net/shenjian58/article/details/93691224