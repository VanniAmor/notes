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



![image-20210729223722019](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210729223722019.png)



LSN是8字节的数字，每个页有LSN，重做日志中也有LSN，Checkpoint也有LSN。

有几中LSN，如下

1. Log sequence number，表示数据库开启到现在已经产生的日志量，即LSN,日志序列号，单位是字节，值越大，说明数据库更新越多。通过它可以计算日志的产生速度。

2. Log flushed up to，表示日志已经刷新到哪个点了，其值 > = LSN

   LSN - Log flushed up to表示log buffer中还有多少日志未刷新到磁盘，如果系统hang住了，可以通过 LSN - Log flushed up to来看下是否是由于log buffer满了导致系统hang住了。一般情况下，如果超过30%的日志还没有刷新到日志文件中，就需要增大**innodb_log_buffer_size**的值。

3. Pages checkPoint at，表示脏页已经刷到哪个点了，表示之前的logfile里的日志可以被覆盖了。

   【Log  flushed up to  - Pages checkPoint at 】表示不可覆盖的脏页的量，如果它的值较少，说明Log flushed up to和Pages flushed up to的值比较接近，表示脏页刷的比较快，可以被覆盖的logfile就多。如果2-3的值大，表示脏页刷新的速度慢，能被覆盖的logfile就少。

4. Last checkpoint at，表示最后一次检查的log的位置。它的值表示系统启动时从哪个点去恢复，redo log做崩溃恢复时指定的起点。去做崩溃恢复时，终点是最新的一条logfile，起点就是checkpoint，记录的最早脏的点。

![image-20210729224442965](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210729224442965.png)



## CheckPoint分类

有两种CheckPoint：

- Share CheckPoint，发生在数据库关闭时将所有脏页都刷新回磁盘，是默认的工作方式，即参数 **innodb_fast_shutdown=1**， 这个过程是阻塞的

- Fuzzy CheckPoint，模糊检查，有以下四种情况

  - master thread checkpoint，以每秒或者每十秒的速度从缓冲池的脏页列表中刷新一定比例的脏页回磁盘，这个过程是异步的，不会阻塞用户线程。

  - flush lru list checkpoint， 通过参数 innodb_lru_scan_depth 控制LRU列表中可用页的数量，发生了这个checkpoint时，说明脏页写入速度过慢。

  - async/sync flush checkpoint，指的是重做日志不可用的情况。当重做日志不可用时，

    如果不能被覆盖的脏页数量（2-3）达到 75%时，触发异步checkpoint。

    不能被覆盖的脏页数量（2-3）达到90%时，同步并且阻塞用户线程，然后根据 flush 列表最早脏的顺序刷脏页。

    当这个事件中的任何一个发生的时候，都会记录到errlog 中，一旦errlog出现这种日志提示，一定需要加大logfile的组数。

  - dirty page too much checkpoint，脏页太多，会发生强制写日志，**会阻塞用户线程**，由【innodb_max_dirty_pages_pct】参数控制，默认为75%



 mysql数据库为了提高事务的操作效率，在事务提交之后并不会立即将修改后的数据写入磁盘，而是通过日志先行（write log ahead）的策略保证事务的持久性。对于脏页，则通过异步的方式刷新到磁盘上，MySQL则是采用了checkpoint技术实现了异步刷新脏页的目的。



## Log Buffer

![image-20210729230653266](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210729230653266.png)



![image-20210731112940757](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731112940757.png)

当在MySQL中对InnoDB表进行更改时，这些更改首先存储在InnoDB日志缓冲区的内存中，然后写入通常称为重做日志（redo logs）的InnoDB日志文件中。

日志缓冲区是内存存储区域，用于保存要写入磁盘上的日志文件的数据。日志缓冲区大小由innodb_log_buffer_size 变量定义，默认大小为16MB。

日志缓冲区的内容定期刷新到磁盘。较大的日志缓冲区可以运行大型事务，而无需在事务提交之前将重做日志数据写入磁盘。因此，如果有更新，插入或删除许多行的事务，则增加日志缓冲区的大小可以节省磁盘I/O。

![image-20210731113311922](C:/Users/99380/AppData/Roaming/Typora/typora-user-images/image-20210731113311922.png)



- innodb_flush_log_at_trx_commit ：控制如何将日志缓冲区的内容写入并刷新到磁盘。（对应WAL的三种方式），默认为1
- innodb_flush_log_at_timeout ：控制日志刷新频率。



【注意】由于进程调度策略问题，“每秒执行一次flush到硬盘的操作”，并不是100%保证的每秒



### 1、如何确定InnoDB日志缓冲区大小是否需要调整？

如果磁盘I/O导致性能问题，则需要观察事务,例如涉及许多BLOB条目的事务。只要InnoDB日志缓冲区已满，便会将其刷新到磁盘，因此增加缓冲区大小可以减少I/O。

另外需要观察的是MySQL的InnoDB重做日志（redo logs）。它们是磁盘上的一组文件，其中包含对InnoDB表所做的所有最近更改的记录，并在崩溃恢复期间使用。根据环境的不同，这些文件可能需要根据默认设置进行调整，例如日志的总数和大小。

日志文件的缺省数量为两个，分别名为 ib_logfile0 和 ib_logfile1 。日志具有固定大小，默认大小取决于MySQL版本。从5.7版本开始，默认值是每个48MB，从MySQL 5.6.3开始，最大总大小为 512GB。

### 2、如何确定日志文件的大小是否需要调整？

如果应用程序是写密集型应用程序，则可以使用48MB，并且鉴于日志以循环方式工作，当日志写满时，有必要对磁盘上的数据文件进行写操作。因此，如果日志文件的大小太小，可能会导致频繁的磁盘写入甚至是等待，从而极大地降低了性能。可以通过查看日志序列号状态变量log_lsn_current和log_lsn_last_checkpoint来观察刷新的频率。通过将两个值相减，并与重做日志空间的总大小进行比较，您可以了解刷新是否比期望的发生更频繁。

> 要调整 innodb_log_buffer_size 或 innodb_log_file_size 变量，必须在MySQL的配置文件中显式定义它们。在进行这些更改之前，请关闭实例，以确保MySQL正确无误地停止运行。如果在关闭过程中出现错误，则现有的日志文件可能尚未完全写入数据文件，并且数据可能会丢失。
>
> 总结：在考虑MySQL数据库性能时，Innodb_log_buffer_size和Innodb_log_file_size是要分析的两个重要变量。如果大型事务导致过多的磁盘I/O，则可以增加InnoDB日志缓冲区。出于相同的原因，InnoDB日志文件通常会基于默认值增加。如果它变得太快，则检查点刷新活动会增加，并且可能会导致性能降低。



## 参考文献

https://my.oschina.net/fileoptions/blog/2988622

https://www.cnblogs.com/chenpingzhao/p/5107480.html

https://www.cnblogs.com/geaozhang/p/7341333.html

https://blog.csdn.net/qq_24432315/article/details/108162809

https://www.cnblogs.com/wy123/p/8353245.html

