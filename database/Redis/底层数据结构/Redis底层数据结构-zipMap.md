dict作为字典结构，优点很多，但是对于字典中的键值均为二进制数据，且长度都很小时，dict中的一堆指针会浪费不少的内存，因此Redis又实现了一个轻量级的字典结构 -- zipMap



**zipmap适合使用的场合是:** 

- 键值对量不大

- 单个键，单个值长度小

需要注意的是，dict字典本身不持有数据，而是持有dictEntry节点的指针（数据存放在dictEntry）。而zipMap是直接持有数据的。

## zipMap内存布局

而整个zipmap就是在一块内存区中依次存放key-value字符串。

![image-20210121201848222](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210121201848222.png)

（1）**zmlen**

​	 一个字节（8位），记录当前zipmap中的key-value对的数量。由于zmlen只有8位，故其表示的数量只能为 0~254（zipMap结束符固定为 0xFF，即255），当节点数大于254时，则需要遍历整个zipmap来获取key-value对的数量

(2) **key length 和 value length**:

​	这两个length字段的编码方式和ziplist有些相似。能够用**1个字节或5个字节**编码表示，详细例如以下：

- 假设随后的字符串（key或value）长度小于或等于253，直接用1个字节表示其长度。

- 假设随后的字符串（key或value）长度超过254。则用5个字节表示，当中第一个字节值为254，接下来的4个字节才是字符串长度。

（3）**free**

​		一个字节，表示随后的value后面的空闲字节数，假设zipmap存在”foo” => “bar”这样一个键值对，随后我们将“bar”设置为“hi”。此时free = 1，表示value字符串后面有1个字节大小的空暇空间。

​		free字段是一个占1个字节的整型数，它的值一般都比較小。假设空暇区间太大，zipmap会进行调整以使整个map尽可能小，这个过程发生在zipmapSet操作中。

（4）**end字段**

​		end是zipMap的结束符，占用一个字符

## zipMap操作



### 创建空的zipMap

```c
unsigned char *zipmapNew(void) {
    // 初始化时仅仅有2个字节,第1个字节表示zipmap保存的key-value对的个数。第2个字节为结尾符
    unsigned char *zm = zmalloc(2);

    // 当前保存的键值对个数为0
    zm[0] = 0; /* Length */
    zm[1] = ZIPMAP_END;
    return zm;
}
```

### 查询操作

key-value是顺序存放的，所以仅能通过遍历整个zipMap来查找目标键值对。但是每个key-value中都记录了自己的长度，所以查询速度也没有想象中的慢

```c
 /* 按关键字key查找zipmap，假设totlen不为NULL，函数返回后存放zipmap占用的字节数 */
static unsigned char *zipmapLookupRaw(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned int *totlen) {
    // zipmap中第1个字节是zmlen字段。zm+1跳过第1个字节
    unsigned char *p = zm+1, *k = NULL;
    unsigned int l,llen;

    // 从前往后查找
    while(*p != ZIPMAP_END) {
        unsigned char free;

        /* Match or skip the key */
        // 确定key字符串的长度
        l = zipmapDecodeLength(p);
        // 确定保存key字符串长度所须要的字节数，也就是len字段所须要的字节数
        llen = zipmapEncodeLength(NULL,l);
        // 比較当前key与给定key是否匹配
        if (key != NULL && k == NULL && l == klen && !memcmp(p+llen,key,l)) {
            /* Only return when the user doesn't care
             * for the total length of the zipmap. */
            // 假设totlen为NULL。表示函数调用者不关心zipmap占用的字节数，此时直接返回p，否则先记录下p指针然后继续遍历
            if (totlen != NULL) {
                k = p;
            } else {
                return p;
            }
        }
        // p加上llen和l。到了value节点处
        p += llen+l;
        /* Skip the value as well */
        // 确定value字符串的长度
        l = zipmapDecodeLength(p);
        // 确定保存value字符串长度所须要的字节数，也就是len字段所须要的字节数
        p += zipmapEncodeLength(NULL,l);
        // 读出free字段的值（前面我们讲过：free仅仅占用一个字节）
        free = p[0];
        // 跳到下一个key节点的
        p += l+1+free; /* +1 to skip the free byte */
    }
    // 到这里遍历完整个zipmap。得到其占用的字节数
    if (totlen != NULL) *totlen = (unsigned int)(p-zm)+1;
    return k;
}
```

### set操作

```c
unsigned char *zipmapSet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char *val, unsigned int vlen, int *update) {
    unsigned int zmlen, offset;
    unsigned int freelen, reqlen = zipmapRequiredLength(klen,vlen);
    unsigned int empty, vempty;
    unsigned char *p;

    freelen = reqlen;
    if (update) *update = 0;
    p = zipmapLookupRaw(zm,key,klen,&zmlen); //首先在zipmap中查找key
    if (p == NULL) {
        //当zipmap中不存在key时，扩展内存
        zm = zipmapResize(zm, zmlen+reqlen);
        p = zm+zmlen-1;
        zmlen = zmlen+reqlen;

        if (zm[0] < ZIPMAP_BIGLEN) zm[0]++;
    } else {  //zipmap中已有key，则需要将其value更新为val
        if (update) *update = 1;
        freelen = zipmapRawEntryLength(p);
        if (freelen < reqlen) { //如果空闲空间不足时，需要扩展内存
            offset = p-zm;
            zm = zipmapResize(zm, zmlen-freelen+reqlen);
            p = zm+offset;
            memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));
            zmlen = zmlen-freelen+reqlen;
            freelen = reqlen;
        }
    }

    //将当前key-value对后面的内容向后移动，预留空间
    empty = freelen-reqlen;
    if (empty >= ZIPMAP_VALUE_MAX_FREE) { 
        offset = p-zm;
        memmove(p+reqlen, p+freelen, zmlen-(offset+freelen+1));
        zmlen -= empty;
        zm = zipmapResize(zm, zmlen);
        p = zm+offset;
        vempty = 0;
    } else {
        vempty = empty;
    }

    //向zipmap中写入key-value对
    p += zipmapEncodeLength(p,klen);
    memcpy(p,key,klen);
    p += klen;
    p += zipmapEncodeLength(p,vlen);
    *p++ = vempty;
    memcpy(p,val,vlen);
    return zm;
}
```

### get操作

返回1时表示查找成功。当查找成功时，将value的地址和value_len分别保存在value和vlen中返回。

```c
int zipmapGet(unsigned char *zm, unsigned char *key, unsigned int klen, unsigned char **value, unsigned int *vlen) {
    unsigned char *p;

    if ((p = zipmapLookupRaw(zm,key,klen,NULL)) == NULL) return 0;
    p += zipmapRawKeyLength(p);
    *vlen = zipmapDecodeLength(p);
    *value = p + ZIPMAP_LEN_BYTES(*vlen) + 1;
    return 1;
}
```