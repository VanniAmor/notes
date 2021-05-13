## 旧版的缺憾



先来捋一下先前的算法的步骤或要点

- slot_state中的低位表示下一个空闲字节，初始为0
- 某个线程原子的在slot中声明一个位置，谓之【连接】，可以通过对一个索引变量进行CAS操作来实现。
- 索引与声明的总字节数相等，因此我们称之为**连接**计数器（**join** counter）。
- **leader线程（即第一个连接slot的线程，其slot_offset=0）锁住slot池，并准备新的slot以方便后续的线程写入，然后切换其所在的slot的slot_state为COPY**
- 所有连接到该slot的线程，在其slot_state还不是COPY的时候，必须进行自旋等待
- 当slot切换为COPY状态后，已连接的线程可以并行进行复制操作
- 直到所有连接slot的线程都实际执行了它们的复制操作，slot才能被写入操作系统，因此slot必须要跟踪线程何时完成复制。
- 可以在**释放**计数器（**release** counter）上进行一个原子操作来跟踪写入的字节。当它的**release** == **join**时，一个slot就可以写入操作系统了。
- 还必须跟踪**slot**的**状态**，这样线程就知道它何时不可用、何时可连接、以及何时准备写入操作系统等。



为了保证安全地向操作系统写入一个slot，线程必须确定slot的状态是否【允许】，并且所有已在其缓冲区进行声明（**join**）的线程都已完成数据的复制（**release**）—— 而且这三个组件的检查必须原子地执行。



而在旧版的算法中，将slot的状态，join counter，release counter 都使用了一个变量来存储，就是 **slot_state**，而采取这种做法的原因：

- 为了实现以上说的原子操作，先确认slot的状态，然后线程join，最后进行复制
- 在于CPU不允许涉及两个以上寄存器的原子操作（理论上来说可以的）。



而该算法的缺点，就是在leader线程完成slot_state的更改之前，必须进行busy-wait（通过自旋等待来实现），这样当有大量工作线程的时候，会严重损耗CPU性能。这样就造成了logJam



## 新的算法

新的算法思路，就是为线程的join和release维护两个独立的计数器，来达到消除线程等待的需要。

在不需要等待连接/释放阶段更替的情况下，线程可以声明一个点并对其进行写入，记录写入的字节然后离开，过程中无需任何等待。这个实现消除了对“leader”线程的需求，并将职责分摊到了两个线程上。当一个线程在执行填充缓冲区的连接时，它会关闭这个slot并准备一个新的。当一个线程在没有其它等待的写入操作并且缓冲区已满的情况下完成释放时，它会将缓冲区写入操作系统。



​	



以下用图片的方式来展示这个算法

- 设立两个计数器，join，release，均为32位，所以可以用一个int64来存储，这样从逻辑上将一个寄存器分割为维护所有必要信息的部分：slot的状态和两个计数器。

- 当slot的threshold到达阈值时，其最后一个进行CAS操作的线程将slot的state设置为CLOSED，表明此slot应该被关闭并准备写入操作系统



```
有四个线程准备往slot中写入数据，假设单个slot的threshold为1280（实际上会比这个值更大）

[![slide-00](http://mongoing.com/wp-content/uploads/2019/01/slide-00-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-00.png)



```
绿色线程通过读取当前的slot->state开始其连接，其中JOINED计数和RELEASED计数均为零。
```

[![slide-01](http://mongoing.com/wp-content/uploads/2019/01/slide-01-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-01.png)



```
绿色线程通过将其大小256添加到JOINED位来计算new_state。
```

[![slide-02](http://mongoing.com/wp-content/uploads/2019/01/slide-02-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-02.png)



```
绿色线程执行CAS原子操作。它以旧的JOINED计数0作为偏移量，将slot->state的JOINED计数更新为256。
```

[![slide-03](http://mongoing.com/wp-content/uploads/2019/01/slide-03-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-03.png)



```
无需等待，绿色线程就将数据复制到其偏移量0。
```

[![slide-04](http://mongoing.com/wp-content/uploads/2019/01/slide-04-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-04.png)



```
绿色线程通过再次读取当前的slot->state以开始释放操作。
这是必须的，因为尽管这里没有描述，但实际上其它线程可能在这期间已经更新了slot->state。
```

[![slide-05](http://mongoing.com/wp-content/uploads/2019/01/slide-05-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-05.png)



```
现在，线程将其大小256添加到RELEASED位以创建new_state。
```

[![slide-06](http://mongoing.com/wp-content/uploads/2019/01/slide-06-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-06.png)



```
通过CAS原子操作，用新的RELEASED计数256来更新slot->state。
```

[![slide-07](http://mongoing.com/wp-content/uploads/2019/01/slide-07-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-07.png)



```
绿色线程重新开始工作，而蓝色线程开始连接。没有发生锁或等待！
```

[![slide-08](http://mongoing.com/wp-content/uploads/2019/01/slide-08-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-08.png)



```
蓝色线程通过读取当前的slot->state开始其连接，其中JOINED和RELEASED计数均为256。
```

[![slide-09](http://mongoing.com/wp-content/uploads/2019/01/slide-09-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-09.png)



```
蓝色线程通过将其大小256添加到JOINED位来计算new_state。
```

[![slide-10](http://mongoing.com/wp-content/uploads/2019/01/slide-10-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-10.png)



```
蓝色线程执行一个CAS原子操作，用旧的JOINED计数256作为偏移量，将slot->state的JOINED计数更新为512。
```

[![slide-11](http://mongoing.com/wp-content/uploads/2019/01/slide-11-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-11.png)



```
在蓝色线程完成复制和释放之前，紫色线程开始连接。
```

[![slide-12](http://mongoing.com/wp-content/uploads/2019/01/slide-12-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-12.png)



```
紫色线程读取当前的slot->state，这是这个过程中第一次在一个连接操作开始时JOINED位（512）和RELEASED位（256）的计数产生了不同。
```

[![slide-13](http://mongoing.com/wp-content/uploads/2019/01/slide-13-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-13.png)



```
紫色线程通过将其大小128添加到JOINED位来计算new_state。
```

[![slide-14](http://mongoing.com/wp-content/uploads/2019/01/slide-14-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-14.png)



```
紫色线程执行一个CAS原子操作，用旧的JOINED计数512作为偏移量，将slot->state的JOINED计数更新为640。
```

[![slide-15](http://mongoing.com/wp-content/uploads/2019/01/slide-15-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-15.png)



```
紫色线程将数据复制到其偏移量512。
```

[![slide-16](http://mongoing.com/wp-content/uploads/2019/01/slide-16-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-16.png)



```
紫色线程通过读取当前的slot->state以开始释放操作。
```

[![slide-17](http://mongoing.com/wp-content/uploads/2019/01/slide-17-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-17.png)



```
紫色线程将其大小128添加到RELEASED位以创建new_state。
```

[![slide-18](http://mongoing.com/wp-content/uploads/2019/01/slide-18-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-18.png)



```
与此同时，蓝色线程可以并行地将其数据复制到偏移量256。
```

[![slide-19](http://mongoing.com/wp-content/uploads/2019/01/slide-19-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-19.png)



```
回来看紫色线程：通过CAS原子操作，用新的RELEASED计数384来更新slot->state。
```

[![slide-20](http://mongoing.com/wp-content/uploads/2019/01/slide-20-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-20.png)



```
现在紫色线程已经完成了任务并重新开始工作。
```

[![slide-21](http://mongoing.com/wp-content/uploads/2019/01/slide-21-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-21.png)



```
同时，蓝色线程开始释放，读取当前的slot->state，JOINED计数为640，RELEASED计数为384。
```

[![slide-22](http://mongoing.com/wp-content/uploads/2019/01/slide-22-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-22.png)



```
蓝色线程通过将其大小256添加到RELEASED位来计算new_state。
```

[![slide-23](http://mongoing.com/wp-content/uploads/2019/01/slide-23-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-23.png)



```
紫色线程执行一个CAS原子操作，用新的RELEASED计数640来更新slot->state，现在JOINED和RELEASED计数再次相等了。
```

[![slide-24](http://mongoing.com/wp-content/uploads/2019/01/slide-24-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-24.png)



```
蓝色线程重新开始工作。
```

[![slide-25](http://mongoing.com/wp-content/uploads/2019/01/slide-25-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-25.png)



```
红色线程开始连接。
```

[![slide-26](http://mongoing.com/wp-content/uploads/2019/01/slide-26-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-26.png)



```
当红色线程计算新的JOINED计数时发现它的数据将越过缓冲区阈值，这表示slot应该被关闭并准备写入操作系统了。
```

[![slide-27](http://mongoing.com/wp-content/uploads/2019/01/slide-27-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-27.png)



```
当红色线程执行其CAS操作时，还将相应的位进行添加并将slot状态置为CLOSED。
红色线程以旧的JOINED计数640作为偏移量，设置slot->state的JOINED计数为1664。
```

[![slide-28](http://mongoing.com/wp-content/uploads/2019/01/slide-28-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-28.png)



```
红色线程执行复制操作。
```

[![slide-29](http://mongoing.com/wp-content/uploads/2019/01/slide-29-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-29.png)



```
红色线程执行释放操作，并将slot->state的RELEASED计数设置为1664。
```

[![slide-30](http://mongoing.com/wp-content/uploads/2019/01/slide-30-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-30.png)



```
当 STATE == DONE 并且 JOINED == RELEASED 时，红色线程知道所有复制均已完成，可以安全地将缓冲区写入操作系统了。
```

[![slide-31](http://mongoing.com/wp-content/uploads/2019/01/slide-31-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-31.png)



```
红色线程发出写操作请求，并重新开始工作。
```

[![slide-32](http://mongoing.com/wp-content/uploads/2019/01/slide-32-1024x640.png)](http://mongoing.com/wp-content/uploads/2019/01/slide-32.png)







## 参考文献

- https://blog.csdn.net/baijiwei/article/details/89739781
- https://www.cnblogs.com/danhuangpai/p/11022789.html
- https://www.cnblogs.com/igoodful/p/13954888.html

- https://mongoing.com/archives/9371
- https://blog.csdn.net/yuanrxdu/article/details/78339295
- https://cloud.tencent.com/developer/article/1405902
- https://cloud.tencent.com/developer/article/1405901?from=article.detail.1405902