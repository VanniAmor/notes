![img](https://img2020.cnblogs.com/blog/565213/202005/565213-20200530222026232-2083340759.png)



mysql作为一个存储系统，具有缓冲池（Buffer Pool）机制，以避免每次查询都进行磁盘IO



## Buffer Pool机制

在计算机中，存在着大名鼎鼎的 【局部性原理】，所以会有mysql中会有预读机制，即把磁盘中的数据先加在到内存中，避免频繁的磁盘IO，提高性能

而在InnoDB中，页大小为16Kb。Buffer Pool的默认大小为128MB。

Buffer Pool中的存储单元也是16Kb，即一个页的大小



Mysql每次的操作，都会先操作Buffer Pool，然后再写入到磁盘



Buffer Pool中需要维护三个链表，来保证其能正常工作

- free 链表，记录Buffer Pool中空闲的页单位
- flush 链表，记录Buffer Pool中哪些数据被修改
- LRU 链表，记录热点数据



Buffer Pool中的page，有三个状态

- free：当前page未被使用
- clean：当前page被使用，对应数据文件中的一个page，但数据未被修改
- dirty：当前page被使用，对应于数据文件中的一个页面，同时页面被修改

clean，dirty两种类型的page，一定位于buf pool的LRU链表中。

与此同时，dirty page还位于buf pool的flush链表中。flush list中的dirty page，按照page的**oldest_modificattion（最晚修改时间）**时间排序，

oldest_modification越大，说明page修改的时间越晚，就排在flush 链表的头部；oldest_modification越小，说明page修改的时间越早，就排在flush链表的尾部。



## FreeList

![图片](https://mmbiz.qpic.cn/mmbiz_png/58iahJ4uK6o3C5iakB529ibxPUQictfnIDJB2MhgOwv9waU37Xtndjh8369ibzAdEubb1cQsmAOns5NNcIFtLicWTSug/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

FreeList中第一个节点为【基节点】，记录整个FreeList的头、尾节点，链表长度等信息



其余的成员节点则指向一个【控制块】，这个控制块为一个指针，指向真正的BufferPool中的页单位数据



**FreeList是为了解决从磁盘中读取数据后，寻找Buffer Pool空闲位置的问题**

**以页为单位，FreeList中记录BufferPool中的空闲位置**



如果这个缓存页中没有存储任何数据，那么它对应的描述信息就会被维护进Free List中。这时当你想把从磁盘中读取出一个数据页放入缓存页中的话，就得先从Free List中找一个节点（Free List中的所有节点都会指向一个从未被使用过的缓存页），那接着就可以把你读取出来的这个数据页放入到该节点指向的缓存页中。

相应的：当数据页中被放入数据之后。它对应的描述信息块会被从Free List中移出。



这里还有一个问题：如何判断从磁盘中读取的数据是否已经在BufferPool中

这个功能的实现依托于另一个数据结构：hash table

**key = 表空间号+数据页号**

**value = 缓存页地址**

如果存在于hash table中，那就说明该数据页已经存在于Buffer Pool中了，优先使用Buffer Pool中的缓存页



## LRUList

### 磁盘预读

根据【局部性原理】，通常都遵循“集中读写”的原则，使用一些数据，大概率会使用附近的数据，所以磁盘读写，**并不是按需读取，而是按页读取**，一次至少读一页数据（一般是4K），如果未来要读取的数据就在页中，就能够省去后续的磁盘IO，提高效率。



### 改良版Lru算法

![img](https://images2015.cnblogs.com/blog/1063598/201703/1063598-20170320102200752-644135341.png)

在mysql中，为了避免快速读取大量数据，如select *而造成的真正热点数据被替代，将整个LRU List设计为两个部分：新生代，老年代

- 当数据从磁盘读入BufferPool中时，首先把数据写入【老年代】
- 当数据在【老年代】存活超过1s（innodb_old_blocks_time参数配置），然后再次被读取时，才会真正写入到【新生代】
- 当【老年代】或【新生代】数据溢出时，会淘汰最旧的数据
- 老年代和新生代占整个Lru链表的比例为（37 ： 63），大约3/8，可通过参数 **innodb_old_blocks_pct** 来配置



## Flush List

为了保证数据一致性，当修改了LruList中的数据时，Mysql是需要将脏页刷到磁盘的，所以此时需要FlushList来记录脏页数据



一旦你对内存中的缓冲页作出了修改，那该缓冲页对应的描述信息块就会添加进 Flush List。这样当Buffer Pool中的数据页不够用时，我们就可以优先将 Flush List中的脏数据页刷新进磁盘中。



### 刷盘时间

- 当MySQL数据库关闭时，会将所有的脏数据页刷新回磁盘。这个功能由参数：`innodb_fast_shutdown=0`控制，默认让InnoDB在关闭前将脏页刷回磁盘，以及清理掉undo log。

- 有一个后台线程Master Thread会按照每秒或者每十秒的速度，异步的将Buffer Pool中一定比例的页面刷新回磁盘中。

- 在MySQL5.7中，Buffer Pool的刷新由page cleaner threads完成。
  - 我们可以通过`innodb_page_cleaners`参数控制page cleaner threads线程的数量，但是当你将这个数值调整的比Buffer Pool的数量还大时，MySQL会自动将 `innodb_page_cleaners`数量设置为`innodb_buffer_pool_instances`的数量。
  - Innodb1.1.x之前需要保证LRU列表中有至少100个空闲页可以使用。低于这个阈值就会触发脏页的刷新。
  - 从MySQL5.6，也就是innodb1.2.X开始，`innodb_lru_scan_depth`参数为每个缓冲池实例指定page cleaner threads 扫描Buffer Pool来查找要刷新的脏页的下行距离。默认为1024，该后台线程每秒都会执行一次。

- 当脏数据页太多时，也会触发将脏数据页刷新回磁盘。该机制可由参数`innodb_nax_dirty_pages_pct`控制，比如将其设置为75，表示，当Buffer Pool中的脏数据页达到整体缓存的75%时，触发刷新的动作。现实情况是该参数默认值为0。以此来禁用Buffer Pool早期的刷新行为。

- 当redo log不可用时，也会强制脏页列表中的脏页刷新回磁盘。这个机制同样由一个后台线程完成。

https://blog.csdn.net/san_yun/article/details/84627490?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_title~default-0.control&spm=1001.2101.3001.4242