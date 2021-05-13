在开篇之前需要理解到，这个Journal日志其实就是WAL日志，在MongoDB中称其为 “journal”



journal日志是实现WiredTiger持久化的关键， 所有的插入、更新操作， 都会在journal里面写入一条日志， 并且在服务器意外退出的时候， 通过checkpoint和该checkpoint之后的日志， 能够迅速的回复服务器的状态。
在Wiredtiger里面， 通过log来表示journal相关的实现， 接下来， 我们看下是如何实现的。



## 写入条件

WiredTiger引擎使用内存缓冲区来保存journal记录，WiredTiger根据以下条件将缓冲的日志记录同步到磁盘

- 从MongoDB 3.2版本开始**每隔50ms**将缓冲的journal同步到磁盘中的journal日志文件
- 如果写入操作设置了j:true，则WiredTiger强制同步日志文件，每次写入时都强制落盘
- 由于MongoDB使用的**journal文件大小限制为100MB**,因此WiredTiger大约每100MB数据创建一个新的日志文件。当WiredTiger创建新的journal文件时，WiredTiger将以前的journal日志文件同步到磁盘中的数据文件中



MongoDB达到上面的提交，便会将更新操作写入日志。这意味着MongoDB会批量地提交更改，即每次写入不会立即刷新到磁盘。不过在默认设置下，系统发生崩溃时，不可能丢失超过50ms的写入数据。



**数据文件默认每60秒刷新到磁盘一次，因此Journal文件只需记录约60s的写入数据。**日志系统为此预先分配了若干个空文件，这些文件存放在/data/db/journal目录中，目录名为_j.0、_j.1等 长时间运行MongoDB后，日志目录中会出现类似_j.6217、_j.6218的文件，这些是当前的日志文件，文件中的数值会随着MongoDB运行时间的增长而增大。数据库正常关闭后，日记文件会被清除(因为正常关闭后就不在需要这些文件了).



> 向mongodb中写入数据是先写入内存，然后每隔60s在刷盘，同样写入journal,也是先写入对应的buffer，然后每隔50ms在刷盘到磁盘的journal文件 使用WiredTiger，即使没有journal功能，MongoDB也可以从最后一个检查点(checkpoint,可以想成镜像)恢复;但是，要恢复在上一个检查点之后所做的更改，还是需要使用Journal



> 上面说的都是针对WiredTiger引擎,对于MMAPv1引擎来说有一点不一样，首先它是每100ms进行刷盘，其次它是通过private view写入journal文件,通过shared view写入数据文件。这里就不过多讲解了，因为MongoDB 4.0已经不推荐使用这个存储引擎了。 从MongoDB 3.2版本开始WiredTiger是MongoDB推荐的默认存储引擎
>
> 参考： https://www.cnblogs.com/danhuangpai/p/11022789.html



在MongoDB的WiredTiger引擎中，Journal的写入方式是 Not Sync，最后再加上 一次Write Only 或 Full Sync

以下是数据写入buffer并落盘到Journal日志的过程

## 多线程无锁并行写入（非常难）

参考文献：

- https://blog.csdn.net/yuanrxdu/article/details/78339295
- https://cloud.tencent.com/developer/article/1405902
- https://cloud.tencent.com/developer/article/1405901?from=article.detail.1405902



开始之前先提几个概念

- CAS算法	https://blog.csdn.net/qq_32998153/article/details/79529704

- 日志操作对象logrec

- wt_log结构，为mongoDB全局的log管理结构

  ```c
  struct wt_log {
       . . .
       active_slot:       # 准备就绪且可以作为合并logrec的slotbuffer对象
  
       slot_pool:         # 系统所有slot buffer对象数组，包括：正在合并的、准备合并和闲置的slot buffer。
  }
  
  ```

  - slot_pool，slot池，池中每个slot对象也可以称为slot buffer对象
  - activit_slot，记录可以进行落盘操作的slot对象

- wt_log_slot结构，即slot_pool中的元素

  ```c
  struct wt_log_slot{
  	...
      state:        # 当前slot的状态,ready/done/written/free这几个状态
  
      buf:          # 缓存合并logrec的临时缓冲区
  
      group_size:        # 需要提交的数据长度
  
      slot_start_offset: # 合并的logrec存入log file中的偏移位置
  
  }
  ```



### 为何是无锁写入呢

MongoDB中每个任务会有大量的线程，当使用互斥体来控制线程的写入时，会造成CPU进行线程的频繁切换，极其耗费资源。

那么不用互斥体来控制的话，又如何避免数据被其他线程覆盖呢，以下是WT引擎的Journal日志写入算法



---

以下为初版算法，即仍有**负伸缩问题**



WiredTiger维护了一个[slot池](https://github.com/wiredtiger/wiredtiger/blob/1c8aa34e3047f315a89d407a637c992e035d05ff/src/include/log.h#L158-L162)。若要将记录写入log，线程将尝试连接当前处于**READY**状态的活动slot。第一步是通过对**slot_state**添加记录的大小来确定剩余的缓冲区空间是否足够。如果够用，线程将通过一个原子的CAS（compare-and-swap）操作对**slot_state**更新总大小。当操作成功时，**slot_state**的先前值指示的是该线程在缓冲区中的偏移量，而新值反映的是下一个要连接线程的正确偏移量（或该缓冲区的最终大小）。

注意此时线程还未真正写入数据！必须等到slot关闭对其它线程的连接，才会在相关联预定的地方执行这些写操作（这个工作会由一个leader线程来完成，后文会说明）。所以现在，它只是在等待并监视**slot_state**，等待其关闭的指示。

如果一个线程获得的偏移量为零，那么它就会成为leader线程。leader线程不会坐等slot关闭，相反它会去关闭slot。但首先，它要从池中准备一个新的slot，以确保可以为后续的连接和写入操作做好准备。在整个过程中，这是一个会用到锁的地方。当这个过程结束时，leader会使用一个原子操作来反置**slot_state**的符号。负的**slot_state**表示**slot**对后续的其它连接已经关闭，这正是其它写入线程正在等待的信号。它们已经知道自己的数据在缓冲区中的位置，因而可以并行地进行复制。完成任务后，它们通过原子地将记录大小添加到**slot_state**来释放slot，该值现在是一个负数，它指代剩余的要复制到缓冲区的总字节数。当**slot_state**变为零时，所有复制均已完成，该slot已完全准备好写入操作系统了。最后那个释放slot的线程会看到这个零值，并负责将缓冲区写入操作系统。而这对其它线程完全没有影响，它们早已继续它们愉快的旅程去了。



```
一个slot正处于READY状态并等待连接线程。一个空闲的slot池既没有被使用也不处于READY状态中等待被连接。
```

[![01](http://mongoing.com/wp-content/uploads/2019/02/01-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/01.png)

```
紫色线程开始连接，在缓冲区中请求128字节的空间。
```

[![02](http://mongoing.com/wp-content/uploads/2019/02/02-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/02.png)

```
Join total是slot_state（0）和紫色线程数据大小（128）的总和。
```

[![03](http://mongoing.com/wp-content/uploads/2019/02/03-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/03.png)

```
CAS原子操作确保只有在slot_state自紫色线程读取以来保持不变的情况下连接才会成功。
```

[![04](http://mongoing.com/wp-content/uploads/2019/02/04-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/04.png)

```
成功：紫色线程声明偏移量“0”，slot_state现在指向下一个可用字节（128）。
```

[![05](http://mongoing.com/wp-content/uploads/2019/02/05-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/05.png)

```
offset == 0 表示紫色线负责准备下一个slot。
```

[![06](http://mongoing.com/wp-content/uploads/2019/02/06-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/06.png)

```
紫色线程对一个互斥体上锁以保护slot池。
```

[![07](http://mongoing.com/wp-content/uploads/2019/02/07-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/07.png)

```
蓝色线程尝试连接，向缓冲区中请求256个字节。
```

[![08](http://mongoing.com/wp-content/uploads/2019/02/08-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/08.png)

```
Join total是slot_state（128）和蓝色线程数据大小（256）的总和。
```

[![09](http://mongoing.com/wp-content/uploads/2019/02/09-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/09.png)

```
CAS原子操作确保old_state依然有效。
```

[![10](http://mongoing.com/wp-content/uploads/2019/02/10-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/10.png)

```
成功：蓝色线程声明偏移量“128”，slot_state现在指向下一个可用字节（384）。
```

[![11](http://mongoing.com/wp-content/uploads/2019/02/11-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/11.png)

```
同时，紫色线程继续准备下一个slot。
```

[![12](http://mongoing.com/wp-content/uploads/2019/02/12-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/12.png)

```
红色线程尝试连接，它有1024字节数据。
```

[![13](http://mongoing.com/wp-content/uploads/2019/02/13-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/13.png)

```
成功：红色线程声明偏移量“384”，slot_state现在指向下一个可用字节（1408）。
```

[![14](http://mongoing.com/wp-content/uploads/2019/02/14-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/14.png)

```
绿色线程尝试连接，它有256字节数据。
```

[![15](http://mongoing.com/wp-content/uploads/2019/02/15-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/15.png)

```
成功：绿色线程声明偏移量“1408”，slot_state现在指向下一个可用字节（1664）。
```

[![16](http://mongoing.com/wp-content/uploads/2019/02/16-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/16.png)

```
紫色线程完成对下一个slot的准备。
```

[![17](http://mongoing.com/wp-content/uploads/2019/02/17-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/17.png)

```
紫色线程将新slot设置为READY状态；而对当前的slot，会通过反转其符号来标识它处于COPY状态。
```

[![18](http://mongoing.com/wp-content/uploads/2019/02/18-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/18.png)

```
紫色线程解锁slot池。已连接的线程可以并行进行复制操作（但为了清晰起见，这些数字将一次一个的显示）。
```

[![19](http://mongoing.com/wp-content/uploads/2019/02/19-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/19.png)

```
蓝色线程将其256字节复制到偏移量128的位置。同时，其它线程连接新的READY状态的slot。
```

[![20](http://mongoing.com/wp-content/uploads/2019/02/20-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/20.png)

```
蓝色线程释放，使用原子操作将它的大小添加至（负值的）slot_state。
```

[![21](http://mongoing.com/wp-content/uploads/2019/02/21-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/21.png)

[![22](http://mongoing.com/wp-content/uploads/2019/02/22-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/22.png)

```
红色线程将其1024字节复制到偏移量384的位置。
```

[![23](http://mongoing.com/wp-content/uploads/2019/02/23-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/23.png)

```
红色线程释放，使用原子操作将它的大小添加至slot_state。
```

[![24](http://mongoing.com/wp-content/uploads/2019/02/24-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/24.png)

[![25](http://mongoing.com/wp-content/uploads/2019/02/25-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/25.png)

```
紫色线程将其数据复制到偏移量0的位置。
```

[![26](http://mongoing.com/wp-content/uploads/2019/02/26-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/26.png)

```
紫色线程释放，将slot_state变为-256。
```

[![27](http://mongoing.com/wp-content/uploads/2019/02/27-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/27.png)

```
绿色线程将其数据复制到偏移量1408的位置。
```

[![28](http://mongoing.com/wp-content/uploads/2019/02/28-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/28.png)

```
绿色线程释放，将slot_state变为0，这意味着其处于“DONE”状态。
```

[![29](http://mongoing.com/wp-content/uploads/2019/02/29-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/29.png)

```
slot_state == 0 说明绿色线程是最后一个释放的线程。
```

[![30](http://mongoing.com/wp-content/uploads/2019/02/30-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/30.png)

```
绿色线程将slot缓冲区写入操作系统。完工！
```

[![31](http://mongoing.com/wp-content/uploads/2019/02/31-1024x720.png)](http://mongoing.com/wp-content/uploads/2019/02/31.png)

## 源码（基本不看）

https://www.cnblogs.com/daizhj/archive/2011/03/21/1990344.html




## 参考文献



- https://blog.csdn.net/baijiwei/article/details/89739781
- https://www.cnblogs.com/danhuangpai/p/11022789.html
- https://www.cnblogs.com/igoodful/p/13954888.html

- https://mongoing.com/archives/9371
- https://blog.csdn.net/yuanrxdu/article/details/78339295
- https://cloud.tencent.com/developer/article/1405902
- https://cloud.tencent.com/developer/article/1405901?from=article.detail.1405902