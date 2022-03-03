预写日志（WAL，Write-Ahead Log）将每次状态更新抽象为一个命令并追加写入一个日志中，这个日志**只追加写入**，也就是顺序写入，所以 IO 会很快。相比于更新存储的数据结构并且更新落盘这个随机 IO 操作，写入速度更快了，并且也提供了一定的持久性，也就是数据不会丢失，可以根据这个日志恢复数据。



![image-20210421203206824](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20210421203206824.png)



从图中可以看出

- 内存的读写速度比磁盘要高出几个数量级
- 磁盘的顺序读写效率堪比内存



## 磁盘读写的瓶颈

参考 《磁盘IO原理.md》

大概整理下就是

即一次访盘请求（读/写）完成过程由三个动作组成：

```text
1）寻道（时间）：磁头移动定位到指定磁道，这部分时间代价最高，最大可达到0.1s左右；
2）旋转延迟（时间）：等待指定扇区旋转至磁头下。与硬盘自身性能有关，xxxx转/分；
3）数据传输（时间）：数据通过系统总线从磁盘传送到内存的时间，一般传输一个字节大概0.02us。
```

而其材质的缘故，也是磁盘读写效率不高的原因之一



## WAL原理



- 通过cache合并多条写操作为一条，减少IO次数
- 日志顺序追加性能远高于数据随机写.
- 随机内存处理性能远高于数据随机处理.

**性能:顺序的日志磁盘处理+随机的数据内存处理>随机的数据磁盘处理**



### 写入过程



1. 一般数据写入过程

![image-20210421210546070](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20210421210546070.png)

- 客户端向数据库发送写命令。
- 数据库收到写命令。
- 数据库通过系统调用将数据写入内核缓冲区（page cache)。
- 操作系统将缓冲区数据传输至磁盘控制器，暂存在磁盘缓冲区。
- 磁盘控制器将数据精准的写入物理磁盘。



2. WAL日志磁盘预写

```text
1.在user buffer的日志中写入”Begin Tran”记录
2.在user buffer的日志页写入要修改的信息
3.在user buffer将要修改的数据写入page cache
4.在user buffer的日志中写入”Commit”记录,将user buffer的日志写入platter
5.发送确认信息到客户端(SMSS,ODBC等）
6.将data page cache写入到磁盘(1.2.3)
```

​	使用 WAL 的数据库系统不会再每执行一条 WAL 操作就将数据刷入数据库文件中，一般积累一定的量然后批量写入，**通常使用页为单位，这是磁盘的写入单位**。 同步页缓存和数据库文件的行为被称为 checkpoint（检查点），一般在 page cache积累到一定页数修改的时候；当然，有些系统也可以手动执行 checkpoint。执行 checkpoint 之后，page cache可以被清空，这样可以保证page cache不会因为太大而性能下降。Checkpoint目的是减少数据库的恢复时间(服务奔溃或重启服务后的恢复)

当系统内存不足时,Lazy writer会自动触发,Lazy writer的目的是保证SQL OS 有空闲缓存块和系统有一定可用内存。



## 耐久模式



WAL根据数据刷盘的频率，可以分为三种耐久性模式



### Full Sync

保证在返回之前将记录刷新到磁盘, 从而使数据能够在**系统级别**的崩溃中幸存下来

即每次写入操作都直接落盘

[![gAwPgA.png](https://z3.ax1x.com/2021/04/30/gAwPgA.png)](https://imgtu.com/i/gAwPgA)



### Write Only

保证记录写入操作系统的文件中, 然后返回给用户, 允许数据在**进程级别**的崩溃后仍然幸存

即每次写入操作都把数据写入到操作系统内核缓存中

[![gABgcd.png](https://z3.ax1x.com/2021/04/30/gABgcd.png)](https://imgtu.com/i/gABgcd)



### No Sync

即将记录保存在内存的缓冲区中, 但不保证将其立即写入文件系统。

[![gABqjs.png](https://z3.ax1x.com/2021/04/30/gABqjs.png)](https://imgtu.com/i/gABqjs)



## 参考文献

https://zhuanlan.zhihu.com/p/228335203

https://blog.csdn.net/hguisu/article/details/7408047

https://cloud.tencent.com/developer/article/1405902

