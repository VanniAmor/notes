# zipList-压缩列表

ziplist是list、hash、zset的底层实现之一，在Redis3.0之后已经不直接用ziplist和linkedlist作为底层实现了，取而代之的是quicklist

这些结构的常规顶层实现如下：

- list：双向链表
- hash：字典dict
- zset：跳表zskiplist

但是当这些结构里面包含的元素较少、并且每个元素要么是小整数或长度较少的字符串时，redis将使用zipList作为这些结构的底层实现。

***ziplist是一个经过特殊编码的双向链表，它的设计目标就是为了提高存储效率。***

## zipList结构

![image-20210120213240209](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210120213240209.png)

zipList是由一系列特殊编码的连续内存块组成的顺序存储结构，类似于数组。**zipList在内存中是连续的**，但是不同于数组，为了节省内存ziplist的每个元素所占的内存大小可以不同（数组中叫元素，ziplist叫节点。）每个节点中可以用来存储一个整数或者一个字符串

- zlbytes：ziplist的长度（单位: 字节)，是一个32位无符号整数
- zltail：ziplist最后一个节点的偏移量，**反向遍历ziplist或者pop尾部节点的时候有用**。
- zlen：ziplist的节点（entry）个数，当这个值小于UINT16_ MAX （65535）时，表示ziplist节点数量，当这个值等于 UINT16_MAX 时， 节点的**真实数量需要遍历整个压缩列表才能计算得出。**
- entry：节点
- zlend：ziplist最后1个字节，是一个结束标记，值固定等于255（0xFF）。

## ziplist节点

普通数组的遍历，因为节点的大小和长度相同，每次移动只需直接指针p+1即可；而ziplist中每个节点大小不同，故此遍历的时候就有点不能直接通过sizeof(entry)

所以zipList将一些必要的偏移信息记录在每一个节点里，使之能跳到上一个或下一个节点

![image-20210120225129777](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210120225129777.png)

每个节点由三个部分组成：

- prevlength：记录上一个节点的长度，为了反向遍历
- encoding：当前节点的编码规则
- data：当前节点的值，可以是数字或字符串

为了节省内存，根据上一个节点的长度prevlength，可以将entry分为两类：

（a）当entry的前八位小于254，则这八位就表示上一个节点的长度

（b）当entyr的前八位等于254，则意味着上一个节点的长度无法用8位表示，后面32位才是真实的prevlength。***用254 不用255(11111111)作为分界是因为255是zlend的值，它用于判断ziplist是否到达尾部***

**根据当前节点存储的数据类型及长度，可以将ziplist节点分为9类**：

### 整形节点六种

节点中的encoding属性占用8位，高两位用来区分整数节点和字符串节点（高2位为11时是整数节点），低6位用来区分节点的类型

```c
#define ZIP_INT_16B (0xc0 | 0<<4)//整数data,占16位（2字节）
#define ZIP_INT_32B (0xc0 | 1<<4)//整数data,占32位（4字节）
#define ZIP_INT_64B (0xc0 | 2<<4)//整数data,占64位（8字节）
#define ZIP_INT_24B (0xc0 | 3<<4)//整数data,占24位（3字节）
#define ZIP_INT_8B 0xfe //整数data,占8位（1字节）
/* 4 bit integer immediate encoding */
//整数值1~13的节点没有data，encoding的低四位用来表示data
#define ZIP_INT_IMM_MASK 0x0f
#define ZIP_INT_IMM_MIN 0xf1    /* 11110001 */
#define ZIP_INT_IMM_MAX 0xfd    /* 11111101 */
```





![image-20210120225903804](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210120225903804.png)

最后一种encoding是存储整数0~12的节点，它没有额外的data部分，encoding的**高4位**表示这个类型，**低4位**就是它的data。

（xxxx不能为1111，否则整个encoding就是1111111，这是整个zipList的结束符），且为了表示数字0，**data是xxxx值 - 1**

### 字符串节点三种

![image-20210120231804315](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210120231804315.png)

- 当data小于63字节时(2^6)，节点存为上图的第一种类型，**高2位**为00，**低6位**表示data的长度。
- 当data小于16383字节时(2^14)，节点存为上图的第二种类型，**高2位**为01，后续14位表示data的长度。
- 当data小于4294967296字节时(2^32)，节点存为上图的第二种类型，**高2位**为10，下一字节起连续32位表示data的长度。

**不同于整数节点encoding永远是8位，字符串节点的encoding可以有8位、16位、40位三种长度**

相同encoding类型的整形节点data大小是固定的，而相同encoding类型的字符串节点，data的长度取决于encoding后半部分的值

```c
#define ZIP_STR_06B (0 << 6)//字符串data,最多有2^6字节(encoding后半部分的length有6位,length决定data有多少字节)
#define ZIP_STR_14B (1 << 6)//字符串data,最多有2^14字节
#define ZIP_STR_32B (2 << 6)//字符串data,最多有2^32字节
```



## 总结

zipList有如下几个特点：

- 节省内存，苟到每一个字节上
- 节点大小不固定，遍历不能直接采用指针，每个节点里面有一个属性`prevLength`来保存 **上一个节点的字节长度**
- 整个zipList的结束标志为 0xFF，即255
- 当节点大于254字节时，其下一个节点的 `prevLength` 无法表示其长度，所以`prevLength`后面32位才是真实的prevlength
- 所以节点拓容升级后，可能会引发后续节点的升级，最坏的情况下，会导致整个 `ziplist` 表中的后续节点拓容
- zipList内部节点是有序的，按照节点的插入顺序进行排序



在大规模数据存储中，zipList几乎不浪费内存空间，苟的高程度甚至达到了每一个字节。也因为如此之苟，其缺点也十分明显：

- 不预留内存空间，在节点移除后，立即进行缩容，即 **每次写操作都会进行内存分配操作**
- 节点的升级，可能会引发整个`zipList` 拓容