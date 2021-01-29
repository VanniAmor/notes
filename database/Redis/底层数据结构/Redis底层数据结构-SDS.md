[toc]



# **SDS 动态字符串**

简单动态字符串（Simple Dynamic String）。Redis的String底层没有使用C语言中的字符串，而是使用了一个自定义的结构SDS来作为默认字符串， SDS在Redis底层中也有大量使用

1. redis 存的数据其实只有两种类型，最基础的数据，一种是字符型，一种是数字型（比如double，long, int)都属于此类。 

2. 所有的存的类型都会有redisObject的方式去存，redisObject是个结构体，具体value会由一个private的指针去指向，且会由相应的encoding 
3. **所有字符型都是用的sds来存储**，这样既能兼容string，又能避免原始string的一些坑，比如/0 做为结束符号这种，sds可以通过他的flag和len 准确知道整个string的长度。
4. set命令中value这个sds的buf[] 会去做压缩。

## 结构定义

在Redis3.2之前，只有一个sdshdr类型，定义如下：

```c
struct sdshdr {
    unsigned int len;//表示sds当前的长度
    unsigned int free;//已为sds分配的长度-sds当前的长度
    char buf[];//sds实际存放的位置
};
```

在Redis3.2分支引入了五种sdshdr类型。

```c
typedef char *sds;

struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

![image-20210113113042835](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210113113042835.png)

![image-20210113160156931](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210113160156931.png)

从代码得知， SDS共有五种不同的头部，其目的是为了满足不同长度的字符串可以使用不同大小的Header，从而节省内存

Header部分主要包含以下几个部分： + len：表示字符串真正的长度，不包含空终止字符 + alloc：表示字符串的最大容量，不包含Header和最后的空终止字符 + flags：表示header的类型。

```
// 五种header类型，flags取值为0~4
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

其中`sdshdr5`并没有实际使用，所以实际上有四种不同的头部

不同类型的头部，用于不同长度sds的存储。

## **SDS 与 C 字符串的区别**

|         | 获取字符串长度           | 缓冲区溢出                                                   | 二进制安全                                                   | C字符串函数                                                  |
| ------- | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| C字符串 | 需要遍历整个字符串，O(n) | 每次操作增加长度时，都需要分配足够的内存空间，否则会导致内存溢出，修改字符串长度N次**必然需要执行N次内存重分配** | 字符串末尾必须是“\0”结尾，即空字符，所以在字符串中不能包含空字符，否则程序会误以为字符串结束。这意味着C字符串只能保存文本数据，不能保存图片、视频、音频等二进制数据 | 可以完全使用<String.h>中的所有函数                           |
| SDS     | 直接访问 len 属性，O(1)  | 采用内存预分配策略，SDS将连续增长N次字符串所需的内存重分配次数从必定N次降低为**最多N次** | SDS并不是靠空字符判断结束，而是**通过len属性来判断结束**，所以buf中保存了什么，读取的就是什么 | 使用部分<String.h>库中的函数，且**遵循C字符串中以空字符结束的惯例** |

## **SDS内存预分配策略**

**当长度小于 1M 时，扩容都是加倍现有的空间，如果超过 1M，扩容时一次只会多扩 1M 的空间**

```
#1.SDS内存分配策略：预分配
（1）如果对SDS字符串修改后，len的值小于1MB，那么程序会分配和len同样大小的空间给free，此时len和free的值是相同。
例如：如果SDS的字符串长度修改为15字节，那么会分配15字节空间给free，SDS的buf属性长度为15（len）+15（free）+1（空字符） = 31字节。
 
（2）如果SDS字符串修改后，len大于等于1MB，那么程序会分配1MB的空间给free。
例如：SDS字符串长度修改为50MB那么程序会分配1MB的未使用空间给free，SDS的buf属性长度为 50MB（len）+1MB（free）+1byte（空字符）。
 
#2.SDS内存释放策略：惰性释放
    当需要缩短SDS字符串时，程序并不立刻将内存释放，而是使用free属性将这些空间记录下来，实际的buf大小不会变，以备将来使用。
```

SDS内部不负责清除未使用的闲置内存空间，因为内部API无法判断这样做的合适时机，即便是操作数据区时导致数据区内存占用减少时，内部API也不会清除闲置内在空间（惰性释放内存），**清除闲置内存空间责任应当由SDS的使用者自行担当**

## **内存不对齐策略**

暂时不能领略其深意

1. 一个32位的cpu， 每一个周期从内存里面能读到32位的数据。
2. 基于这个原因和寄存器的原因，cpu的每次读的地址开始是4的倍数，打个比方我们要读地址2上面长度为4的数据，那么就需要两个周期， cpu首先得从0-3地址上面读数据，然后再从3-7的地址上面的数据， 在这里我们可以看到内存对齐的作用。
   那么问题来了，对于redis 作者一个对内存和cpu 用到极致的作者，为什么还要用非对齐的sds了，原因在于sds的本身结构注定只能非对齐状态。
   请看下图，在对齐状态我们的结构体在内存里面表现形势是如何的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201015161914248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMzMzYxOTc2,size_16,color_FFFFFF,t_70#pic_center)

可以看到不同类型的sds 下面，pad 的位数也是不同的，那么我们要从sds 指针位置访问到flag，在不知道类型的情况下是不可能了，那么有同学又要发问了，去掉sdshdr8的结构不就行了吗，从理论来说这样牺牲的内存也不会太多，也保证了性能，但是这仅仅是在32位系统下面的结构，如果在64位系统，那可能又是另外一个结构了。 好的那么有同学又要说了 我们能不能把指针放到flag开始的位置。答案也是不能，1，这样我们就没办法完美兼容string， 2， 这样我们也会引入各种类型判断调整，所以redis 最后还是用到内存不对齐这个方案。



## SDS编码方式

RedisObject共有10种存储格式

![image-20210114204503009](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210114204503009.png)

SDS两种编码方式

- embstr，当存储小内容时，采用此编码方式
- raw，当存储内容大于64字节时，redis认为是大字符串，使用RAW编码方式

SDS的头部（len，alloc ，flag）需要19个字节，再加上结尾的 \0， 所以embstr最多能够存储44字节，当超过44字节时，redis采用RAW编码方式

![image-20210114212526902](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210114212526902.png)

embstr将RedisObject和SDS对象连续存在一起，使用malloc算法进行分配

而raw需要进行两次malloc，两个对象头在内存上是不连续的。



## 参考资料

- https://www.cnblogs.com/mxxct/p/13857494.html#_label0_2
- https://blog.csdn.net/sinat_35261315/article/details/79028999
- https://www.cnblogs.com/cxy2020/p/13799047.html
- https://blog.csdn.net/u013536232/article/details/105476382/
- https://blog.csdn.net/seky_fei/article/details/106968173
- https://blog.csdn.net/qq193423571/article/details/81637075

- https://blog.csdn.net/weixin_42348880/article/details/112357490
- https://blog.csdn.net/qq_33361976/article/details/109014012