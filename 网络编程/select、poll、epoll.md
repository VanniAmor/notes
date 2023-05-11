## select、poll、epoll三者区别

|     \      |                       select                       |                       poll                       |                            epoll                             |
| :--------: | :------------------------------------------------: | :----------------------------------------------: | :----------------------------------------------------------: |
|  操作方式  |                        遍历                        |                       遍历                       |                             回调                             |
|  底层实现  |                        数组                        |                       链表                       |                            哈希表                            |
|   IO效率   |      每次调用都进行线性遍历，时间复杂度为O(n)      |     每次调用都进行线性遍历，时间复杂度为O(n)     | 事件通知方式，每当fd就绪，系统注册的回调函数就会被调用，将就绪fd放到rdllist里面。时间复杂度O(1) |
| 最大连接数 |             1024（x86）或 2048（x64）              |                      无上限                      |                            无上限                            |
|   fd拷贝   | 每次调用select，都需要把fd集合从用户态拷贝到内核态 | 每次调用poll，都需要把fd集合从用户态拷贝到内核态 |  调用epoll_ctl时拷贝进内核并保存，之后每次epoll_wait不拷贝   |



传统select/poll的致命弱点就是当拥有一个庞大的socket集合时，由于网络的延时，在任意一个时刻只有部分的socket是活跃的，而select/poll每次调用都会进行线性扫描全部的socket集合，导致cpu性能的浪费



## Epoll原理

![img](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/15744422-32af7c797010bf43.jpg)

```c
//用户数据载体
typedef union epoll_data {
   void    *ptr;
   int      fd;
   uint32_t u32;
   uint64_t u64;
} epoll_data_t;
//fd装载入内核的载体
struct epoll_event {
    uint32_t     events;    /* Epoll events */
    epoll_data_t data;      /* User data variable */
};
 //三板斧api
int epoll_create(int size); 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```

可以看到epoll主要的数据结构是双向链表 + 红黑树

每个epoll对象就是一个独立的eventpoll结构体，这个结构体会在内核空间中创造出独立的内存，用于存储使用epoll_ctl方法添加的时间。这些事件会挂到 rbr 红黑树中。

### 主要数据结构

- eventpoll，epoll_create（）创建一个eventpoll数据结构，并分配一个文件描述符与之绑定，因此关闭epoll时需要执行close操作

  ```c
  //核心字段
  struct eventpoll {
      struct mutex mtx;
      // 阻塞在该epoll的进程队列
      wait_queue_head_t wq;
      // 当epoll被另一个epoll监听时需要使用poll_wait记录阻塞在该epoll的队列
      wait_queue_head_t poll_wait;
      // 就绪队列
      struct list_head rdllist;
      rwlock_t lock;
      // 红黑树根节点
      struct rb_root_cached rbr;
      // 记录epitem的单链表
      struct epitem *ovflist;
      struct wakeup_source *ws;
      // 创建该epoll的用户信息
      struct user_struct *user;
      // epoll对应的file
      struct file *file;
  };
  ```

- epitem，epitem是与每个用户态监控IO的fd对应的。

  ```c
  struct epitem {
    //红黑树节点
    struct rb_node  rbn;      
    //双向链表头节点
    struct list_head  rdllink; 
    //ovflist链表中，下一个epitem节点
    struct epitem  *next;
    //事件句柄信息
    struct epoll_filefd  ffd;  
    int  nwait; 
    struct list_head  pwqlist;
    //指向所属的eventpoll
    struct eventpoll  *ep;      
    struct list_head  fllink;   
    //期待发生的事件类型, 预定义的Epoll事件如下
    struct epoll_event  event;  
  };
  ```

### Epoll事件

| 常量        | 说明                     |
| ----------- | ------------------------ |
| EPOLLIN     | 普通或优先级带数据可读   |
| EPOLLRDNORM | 普通数据可读             |
| EPOLLRDBAND | 优先级带数据可读         |
| EPOLLPRI    | 高优先级数据可读         |
| EPOLLOUT    | 普通数据可写             |
| EPOLLWRNORM | 普通数据可写             |
| EPOLLWRBAND | 优先级带数据可写         |
| EPOLLERR    | 发生错误                 |
| EPOLLHUP    | 发生挂起                 |
| EPOLLNVAL   | 描述字不是一个打开的文件 |



当调用epoll_ctl时，会根据当前socket的fd和传入的用户感兴趣的epoll事件，创建一个epitem节点，并放入到eventpoll的红黑树中，其中socket的fd就是红黑树的key。

通过epool_ctl函数添加的epoll事件，都会被放入到红黑树的某个节点内，所以重复对某个socket添加同一个事件，是没有用的。

当完成添加后，该事件会与相应的设备（网卡）驱动程序建立回调关系。当相应的事件发生（比如EPOLLIN），就会调用这个回调函数，该回调函数在内核中被称为**ep_poll_callback**，这个回调函数的作用，其实就是把这个事件添加到eventpoll中rdlist这个双向链表中



当调用epoll_wail时，epoll_wait只需检查rdlist双向链表是否有值（链表中的元素，就是一个个epitem）



## EPOLLONESHOT事件

一个线程在读取完某个socket上的数据后开始处理这些数据，而数据的处理过程中该socket又有新数据可读，此时另外一个线程被唤醒来读取这些新的数据。

在这种情形下，就出现了两个线程同时操作一个socket的局面。这种情况可以使用epoll的EPOLLONESHOT事件实现一个socket连接在任一时刻都被一个线程处理

尽管一个socket在不同事件可能被不同的线程处理，但同一时刻肯定只有一个线程在为它服务，这就保证了连接的完整性，从而避免了很多可能的竞态条件。



【demo】https://blog.csdn.net/weixin_42250655/article/details/100764330



## ET与LT

- LT， level-trigger，水平触发，现在为epoll的默认模式，当一个fd中有数据可读时，每次epoll_wait会一直返回，直到这个fd中可读数据为空
- ET，edge-trigger， 边沿触发，当fd从未就绪变为就绪时，epoll_wait会返回，且只返回一次。



根据以上两点，一般有如下处理

- LT可以使用阻塞或非阻塞操作，且每次读取的字节可根据业务进行调整
- ET只能使用非阻塞操作，否则只会处理最后一个读取到的fd，造成饥饿

- 当传输大文件时，使用 ET + 多次读取fd的形式，避免多次返回fd从内核态切换到用户态
- listen操作，是用的LT模式，当队列中有数据时，一直返回
- connect操作，用的ET模式，当队列中有数据时，返回且仅返回一次



### 应用场景

1. ET模式适用高流量场景，比如nginx就是使用ET模式
2. 一般情况下使用LT模式，比如redis



### DEMO代码

https://www.cnblogs.com/DOMLX/p/9601511.html



## 参考文献

- [epoll原理详解 - 简书 (jianshu.com)](https://www.jianshu.com/p/b0504ae6f645)

- [(42条消息) epoll介绍及原理详解_干锅土鸡的博客-CSDN博客_epoll](https://blog.csdn.net/weixin_46258483/article/details/122562266)
- [(42条消息) epoll详解_一口Linux的博客-CSDN博客_epoll](https://blog.csdn.net/daocaokafei/article/details/117397600)

- https://zhuanlan.zhihu.com/p/272891398

- [Select、Poll、Epoll详解 - 简书 (jianshu.com)](https://www.jianshu.com/p/722819425dbd/)