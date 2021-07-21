![img](https://img2020.cnblogs.com/blog/565213/202005/565213-20200530222026232-2083340759.png)



mysql作为一个存储系统，具有缓冲池（Buffer Pool）机制，以避免每次查询都进行磁盘IO



## Buffer Pool机制

在计算机中，存在着大名鼎鼎的 【局部性原理】，所以会有mysql中会有预读机制，即把磁盘中的数据先加在到内存中，避免频繁的磁盘IO，提高性能

而在InnoDB中，页大小为16Kb。Buffer Pool的默认大小为128MB。

Buffer Pool中的存储单元也是16Kb，即一个页的大小



Mysql每次的操作，都会先把数据写入到BufferPool中，再把数据持久化到磁盘



