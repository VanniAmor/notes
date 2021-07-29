## CheckPoint机制

CheckPoint目的是解决以下几个问题：

1. 缩短数据库的恢复时间；

2. 缓冲池不够用时，将脏页刷新到磁盘；
3. 重做日志不可用时，刷新脏页。



- 当数据库发生宕机时，数据库不需要重做所有的日志，因为Checkpoint之前的页都已经刷新回磁盘。数据库只需对Checkpoint后的重做日志进行恢复，这样就大大缩短了恢复的时间。
- 当缓冲池不够用时，根据LRU算法会溢出最近最少使用的页，若此页为脏页，那么需要强制执行Checkpoint，将脏页也就是页的新版本刷回磁盘。
- 当重做日志出现不可用时，因为当前事务数据库系统对重做日志的设计都是循环使用的，并不是让其无限增大的，重做日志可以被重用的部分是指这些重做日志已经不再需要，当数据库发生宕机时，数据库恢复操作不需要这部分的重做日志，因此这部分就可以被覆盖重用。如果重做日志还需要使用，那么必须强制Checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置。



## LSN值

**对于InnoDB存储引擎而言，是通过LSN（Log Sequence Number）来标记版本的。** LSN表示数据库开启到现在已经产生的日志量。单位是字节，值越大，说明数据库更新越多。通过这个值可以计算日志的产生速度



LSN是8字节的数字，每个页有LSN，重做日志中也有LSN，Checkpoint也有LSN。

有几中LSN，如下

- Log sequence number，表示数据库开启到现在已经产生的日志量，即LSN,日志序列号，单位是字节，值越大，说明数据库更新越多。通过它可以计算日志的产生速度。

- Log flushed up to，表示日志已经刷新到哪个点了，其值 > = LSN

  LSN - Log flushed up to表示log buffer中还有多少日志未刷新到磁盘，如果系统hang住了，可以通过 LSN - Log flushed up to来看下是否是由于log buffer满了导致系统hang住了。一般情况下，如果超过30%的日志还没有刷新到日志文件中，就需要增大**innodb_log_buffer_size**的值。

- Pages checkPoint at，表示最后一次检查点的log位置，





## CheckPoint分类

有两种CheckPoint：

- Share CheckPoint，发生在数据库关闭时将所有脏页都刷新回磁盘，是默认的工作方式，即参数 **innodb_fast_shutdown=1**， 这个过程是阻塞的
- Fuzzy CheckPoint









## 参考文献

https://my.oschina.net/fileoptions/blog/2988622

https://www.cnblogs.com/chenpingzhao/p/5107480.html

https://www.cnblogs.com/geaozhang/p/7341333.html

https://blog.csdn.net/qq_24432315/article/details/108162809

https://www.cnblogs.com/wy123/p/8353245.html
