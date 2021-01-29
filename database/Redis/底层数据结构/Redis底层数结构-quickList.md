

在开始介绍之前，文中会涉及到两个重要配置（redis.conf中）

```c
list-max-ziplist-size -2
list-compress-depth 0
```

## quickList 

![image-20210121141636172](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210121141636172.png)

Redis对外暴露的list数据类型，其底层所依赖的数据结构就是`quickList`

quickList在quicklist.c中解释说 `A doubly linked list of ziplists` 即一个由zipList组成的双向链表

总结来说，quickList特点有如下几点：

- 宏观上来说，是一个双线链表，进行插入或删除操作时虽然时间复杂度为O(n)，但是不需要内存的复制。且访问两端元素复杂度为O(1)
- 微观上，quickList是一片片的zipList，每一个zipList节点内存连续且顺序存储，可以通过二分查找的复杂度O(log(n))进行元素定位

## quikList与zipList

那么为何quickList要如此设计，这大概是空间和时间上的一个折中方案吧：

- 双向链表便于操作两端节点，时间复杂度都是O(1)，但是在每个节点上除了保存数据外，还要额外保存两个指针。其次，双向链表的每个节点都是单独的内存块，地址不连续，容易产生内存碎片。
- zipList是一块连续内存，所以存储效率高，但是其不便于修改操作。且每一次数据变动，都会引发一次内存里的realloc。

于是，结合两者优势，就诞生了quickList，但是一个quickList中包含多少zipList，每个zipList中含有多少个数据呢，这里有两个点

- 每个quicklist节点上的ziplist越短，则内存碎片越多。内存碎片多了，有可能在内存中产生很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个quicklist节点上的ziplist只包含一个数据项，这就蜕化成一个普通的双向链表了。
- 每个quicklist节点上的ziplist越长，则为ziplist分配大块连续内存空间的难度就越大。有可能出现内存里有很多小块的空闲空间（它们加起来很多），但却找不到一块足够大的空闲空间分配给ziplist的情况。这同样会降低存储效率。这种情况的极端是整个quicklist只有一个节点，所有的数据项都分配在这仅有的一个节点的ziplist里面。这其实蜕化成一个ziplist了。

基于这种情况，Redis提供了一个参数 `list-max-ziplist-size`，让开发者根据自身情况进行调整

## quickList相关配置

Redis提供了两个配置项来配置quickList和zipList的关系

### list-max-ziplist-size

- 当数字为负数，表示以下含义：
- -1 每个quicklistNode节点的ziplist字节大小不能超过4kb。（建议）
- -2 每个quicklistNode节点的ziplist字节大小不能超过8kb。（默认配置）
- -3 每个quicklistNode节点的ziplist字节大小不能超过16kb。（一般不建议）
- -4 每个quicklistNode节点的ziplist字节大小不能超过32kb。（不建议）
- -5 每个quicklistNode节点的ziplist字节大小不能超过64kb。（正常工作量不建议）
- 当数字为正数，表示：ziplist结构所最多包含的entry个数。最大值为 215215。

### list-compress-depth

当list很长的时候，链表双端的节点访问频率很高，而中间数据访问频率比较低（访问效率也很差），基于此，Redis提供了另外一个配置，能够把链表中的数据进行压缩，从而节省内存

- 0 表示不压缩。（默认）

- 1 表示quicklist列表的两端各有1个节点不压缩，中间的节点压缩。

- 2 表示quicklist列表的两端各有2个节点不压缩，中间的节点压缩。

- 3 表示quicklist列表的两端各有3个节点不压缩，中间的节点压缩。

- 以此类推，最大为 2^16

  

## 各部分结构实现

### quicklist表头结构

```c
typedef struct quicklist {
    //指向头部(最左边)quicklist节点的指针
    quicklistNode *head;

    //指向尾部(最右边)quicklist节点的指针
    quicklistNode *tail;

    //ziplist中的entry节点计数器
    unsigned long count;        /* total count of all entries in all ziplists */

    //quicklist的quicklistNode节点计数器
    unsigned int len;           /* number of quicklistNodes */

    //保存ziplist的大小，配置文件设定，占16bits
    int fill : 16;              /* fill factor for individual nodes */

    //保存压缩程度值，配置文件设定，占16bits，0表示不压缩
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```

### quicklist节点

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;     //前驱节点指针
    struct quicklistNode *next;     //后继节点指针

    //不设置压缩数据参数recompress时指向一个ziplist结构
    //设置压缩数据参数recompress指向quicklistLZF结构
    unsigned char *zl;

    //压缩列表ziplist的总长度
    unsigned int sz;                  /* ziplist size in bytes */

    //ziplist中包的节点数，占16 bits长度
    unsigned int count : 16;          /* count of items in ziplist */

    //表示是否采用了LZF压缩算法压缩quicklist节点，1表示压缩过，2表示没压缩，占2 bits长度
    unsigned int encoding : 2;        /* RAW==1 or LZF==2 */

    //表示一个quicklistNode节点是否采用ziplist结构保存数据，2表示压缩了，1表示没压缩，默认是2，占2bits长度
    unsigned int container : 2;       /* NONE==1 or ZIPLIST==2 */

    //标记quicklist节点的ziplist之前是否被解压缩过，占1bit长度
    //如果recompress为1，则等待被再次压缩
    unsigned int recompress : 1; /* was this node previous compressed? */

    //测试时使用
    unsigned int attempted_compress : 1; /* node can't compress; too small */

    //额外扩展位，占10bits长度
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```



### 压缩后的ziplist结构

当指定使用lzf压缩算法压缩ziplist的entry节点时，quicklistNode结构的zl成员指向quicklistLZF结构

```c
typedef struct quicklistLZF {
    //表示被LZF算法压缩后的ziplist的大小
    unsigned int sz; /* LZF size in bytes*/

    //保存压缩后的ziplist的数组，柔性数组
    char compressed[];
} quicklistLZF;
```



### zipList信息管理结构quickListEntry

和zipList一样，entry结构在储存时是一连串的内存块，需要将其每个entry节点的信息读取到管理该信息的结构体中，以便操作。在quicklist中定义了自己的结构。

```c
//管理quicklist中quicklistNode节点中ziplist信息的结构
typedef struct quicklistEntry {
    const quicklist *quicklist;   //指向所属的quicklist的指针
    quicklistNode *node;          //指向所属的quicklistNode节点的指针
    unsigned char *zi;            //指向当前ziplist结构的指针
    unsigned char *value;         //指向当前ziplist结构的字符串vlaue成员
    long long longval;            //指向当前ziplist结构的整数value成员
    unsigned int sz;              //保存当前ziplist结构的字节数大小
    int offset;                   //保存相对ziplist的偏移量
} quicklistEntry;
```

## 参考文献

https://blog.csdn.net/men_wen/article/details/70229375

https://www.cnblogs.com/neooelric/p/9621736.html

http://zhangtielei.com/posts/blog-redis-quicklist.html