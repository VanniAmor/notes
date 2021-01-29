## intset-整数集合

【博客】https://blog.csdn.net/zgaoq/article/details/89708500

intset是Redis-set的底层实现之一，**当存储整数集合并且数据量较小**的情况下Redis会使用intset作为set的底层实现。当数据量较大或者集合元素为String时，则会使用dict来实现



首先回顾下Set的特点：

- 无序，存放在Set中的元素是没有顺序的
- 唯一，Set中的元素不允许重复



而作为Set的实现之一的intSet，除了元素唯一的特点外，元素在intSet内部却是有序的（从小到大）

intSet会把元素按顺序存储在一个数组里面，并通过二分法降低元素查找的时间复杂度。当数据量大时，依赖于查找的Set命令（如SISMEMBER）就会由于O(log(n))的时间复杂度而遇到性能瓶颈，所以当数据量大的时候采用dict代替intSet.

但是当数据量小的时候，O(log(n)) 和 O(1) 的速度是相差不大，且intSet更节省内存。这就是intSet的意义

### intSet结构声明

```c
typedef struct intset {
    uint32_t encoding; //intset的类型编码
    uint32_t length; //成员元素的个数
    int8_t contents[];//用来存储成员的柔性数组
}
// encoding的三种值
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

需要注意的是，contents虽然被声明为 int8_t, 但里面存的成员并不是int8_t类型。intSet成员是什么类型完全取决于encoding变量的值。

虽然每个成员的**“实际类型”**是int8_t，**无法直接通过contents[x]取出索引为x的成员元素**，但是intset.c里提供了些函数，可以按照不同的encoding方式设置/取出contents的成员。（用指针设置，memcpy取出）

如果老老实实通过contents[x]的方式赋值取值，我们就不需要考虑这个字节序的问题（由CPU和操作系统搞定）。由于这种方法在内存上暴力地赋值和取值，所以希望元素在不同机器上存储的字节序一致。但是不同的处理器在内存中存放数据的方式不一定相同，主要分为大端字节和小端字节。为了避免出现数据不统一的情况，**intSet统一采用小端字节存储成员变量**

【拓展】

![image-20210120154008555](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210120154008555.png)

- **大端（big endian）**：在内存中，数据的高字节保存在内存的低地址中，而数据的低字节，保存在内存的高地址中。 **（数据高字节->内存低地址）**
- **小端（little endian）**：在内存中，数据的高字节保存在内存的高地址中，而数据的低字节保存在内存的低地址中。 **（数据低字节->内存低地址）**

且提供了大小端的转换，如下

```c
/* variants of the function doing the actual convertion only if the target
 * host is big endian */
#if (BYTE_ORDER == LITTLE_ENDIAN)
#define memrev16ifbe(p)
#define memrev32ifbe(p)
#define memrev64ifbe(p)
#define intrev16ifbe(v) (v)
#define intrev32ifbe(v) (v)
#define intrev64ifbe(v) (v)
#else
#define memrev16ifbe(p) memrev16(p)
#define memrev32ifbe(p) memrev32(p)
#define memrev64ifbe(p) memrev64(p)
#define intrev16ifbe(v) intrev16(v)
#define intrev32ifbe(v) intrev32(v)
#define intrev64ifbe(v) intrev64(v)
#endif
 
//翻转
void memrev64(void *p) {
    unsigned char *x = p, t;
 
    t = x[0];
    x[0] = x[7];
    x[7] = t;
    t = x[1];
    x[1] = x[6];
    x[6] = t;
    t = x[2];
    x[2] = x[5];
    x[5] = t;
    t = x[3];
    x[3] = x[4];
    x[4] = t;
}
uint64_t intrev64(uint64_t v) {
    memrev64(&v);
    return v;
}
```

## intSet操作

### 赋值、取值操作

```c
/* Set the value at pos, using the configured encoding. */
//按照intset的encoding设置指定位置pos的值
static void _intsetSet(intset *is, int pos, int64_t value) {
    uint32_t encoding = intrev32ifbe(is->encoding);
 
    if (encoding == INTSET_ENC_INT64) {
        ((int64_t*)is->contents)[pos] = value;
        //大端机器在设置contents时将数据按字节翻转，按照小端序存储
        memrev64ifbe(((int64_t*)is->contents)+pos);
    } else if (encoding == INTSET_ENC_INT32) {
        ((int32_t*)is->contents)[pos] = value;
        memrev32ifbe(((int32_t*)is->contents)+pos);
    } else {
        ((int16_t*)is->contents)[pos] = value;
        memrev16ifbe(((int16_t*)is->contents)+pos);
    }
}
/* Return the value at pos, using the configured encoding. */
//按照intset的encoding取出指定位置pos的值
//不对pos进行越界判断，可能会导致undefined behavior
static int64_t _intsetGet(intset *is, int pos) {
    return _intsetGetEncoded(is,pos,intrev32ifbe(is->encoding));
}
/* Return the value at pos, given an encoding. */
//以enc为编码取出整数集合is在pos索引上的值
//不对pos进行越界判断，可能会导致undefined behavior
static int64_t _intsetGetEncoded(intset *is, int pos, uint8_t enc) {
    int64_t v64;
    int32_t v32;
    int16_t v16;
 
    if (enc == INTSET_ENC_INT64) {
        //将contents在pos位置的值赋给v64
        //不能直接写contents[pos]的原因是contents时int8_t类型的，contents[pos]表示是以sizeof(int8_t)为单位移动的指针，而实际的编码是INTSET_ENC_INT64，先将contents指针的类型变为int64_t*
        memcpy(&v64,((int64_t*)is->contents)+pos,sizeof(v64));
        //大端机器在取出contents时将原本按照小端序存储的数据按字节翻转，读出正确的值
        memrev64ifbe(&v64);
        return v64;
    } else if (enc == INTSET_ENC_INT32) {
        memcpy(&v32,((int32_t*)is->contents)+pos,sizeof(v32));
        memrev32ifbe(&v32);
        return v32;
    } else {
        memcpy(&v16,((int16_t*)is->contents)+pos,sizeof(v16));
        memrev16ifbe(&v16);
        return v16;
    }
}
```

### 创建空的intSet

空的intSet的encoding默认是INTSET_ENC_INT16， contents每个成员的逻辑类型是int16_t（虽然还没有成员）

```c
/* Create an empty intset. */
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);
    is->length = 0;
    return is;
}
```

### 查询成员

通过二分法查找成员，若找到，则函数返回1，否则返回0；

而函数传递的参数pos，则是目标成员的位置，当存在时是目标成员在contents[]的位置，不存在时是能被插入的位置

```c
static uint8_t intsetSearch(intset *is, int64_t value, uint32_t *pos) {
    int min = 0, max = intrev32ifbe(is->length)-1, mid = -1;
    int64_t cur = -1;
 
    /* The value can never be found when the set is empty */
    if (intrev32ifbe(is->length) == 0) {
        if (pos) *pos = 0;
        return 0;
    } else {
        /* Check for the case where we know we cannot find the value,
         * but do know the insert position. */
        //判断是否大于小于边界值
        if (value > _intsetGet(is,intrev32ifbe(is->length)-1)) {
            if (pos) *pos = intrev32ifbe(is->length);//value可以被插入的位置
            return 0;
        } else if (value < _intsetGet(is,0)) {
            if (pos) *pos = 0;
            return 0;
        }
    }
 
    //二分
    while(max >= min) {
        // 这里获取中间值，可能会引起内存越界
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }
 
    if (value == cur) {
        if (pos) *pos = mid;//找到了
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
}
```

需要注意的是，当数据量较大时，二分法查询中 mid = (max + min) / 2 可能会越界，

可以改为下面的方式

```c
while (max >= min) {
    mid = min + (max - min)/2;
    ....
    ....
}
```



### 插入操作

当插入成员时，会做以下判断

- 计算value的encoding

- 若value的encoding大于要插入的intSet的encoding，则升级intSet的encoding，并把新的value插入到首部或尾部

- 若value的encoding小于/等于要插入的intSet的encoding，则不需要升级intSet的encoding，调用`intsetSearch`找到合适的插入位置，并插入数据到指定位置，**该位置后的数据都右移一位**

  ```c
  /* Insert an integer in the intset */
  //success传null进来则说明外层调用者不需要知道是否插入成功(value是否已存在)，否则success用于此目的
  intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
      uint8_t valenc = _intsetValueEncoding(value);//根据value的大小计算value的encoding
      uint32_t pos;
      if (success) *success = 1;
   
      /* Upgrade encoding if necessary. If we need to upgrade, we know that
       * this value should be either appended (if > 0) or prepended (if < 0),
       * because it lies outside the range of existing values. */
      if (valenc > intrev32ifbe(is->encoding)) {
          //这种插入需要改变encoding(不需要search，因为encoding改变说明value一定插入在contents首部或者尾部)
          /* This always succeeds, so we don't need to curry *success. */
          return intsetUpgradeAndAdd(is,value);
      } else {
          /* Abort if the value is already present in the set.
           * This call will populate "pos" with the right position to insert
           * the value when it cannot be found. */
          if (intsetSearch(is,value,&pos)) {
              if (success) *success = 0;//intset里已存在该值，返回失败
              return is;
          }
   
          is = intsetResize(is,intrev32ifbe(is->length)+1);
          if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);//右移一格
      }
   
      _intsetSet(is,pos,value);//插入值
      is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
      return is;
  }
   
  /* Return the required encoding for the provided value. */
  //根据v值的大小决定需要的编码类型
  static uint8_t _intsetValueEncoding(int64_t v) {
      if (v < INT32_MIN || v > INT32_MAX)
          return INTSET_ENC_INT64;
      else if (v < INT16_MIN || v > INT16_MAX)
          return INTSET_ENC_INT32;
      else
          return INTSET_ENC_INT16;
  }
   
  /* Upgrades the intset to a larger encoding and inserts the given integer. */
  //这个函数执行的前提是value参数的大小超过了当前编码
  //为is->content重新分配内存并修改编码添加value进这个intset
  static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
      uint8_t curenc = intrev32ifbe(is->encoding);//当前编码类型
      uint8_t newenc = _intsetValueEncoding(value);//新的编码类型
      int length = intrev32ifbe(is->length);
      int prepend = value < 0 ? 1 : 0;//因为value一定超过了编码的限制，所以看value是大于0还是小于0以此决定value放置在content[0]还是content[length]
   
      /* First set new encoding and resize */
      is->encoding = intrev32ifbe(newenc);
      is = intsetResize(is,intrev32ifbe(is->length)+1);
   
      /* Upgrade back-to-front so we don't overwrite values.
       * Note that the "prepend" variable is used to make sure we have an empty
       * space at either the beginning or the end of the intset. */
      while(length--)
          //以curenc为编码倒序取出所有值并赋值给新的位置
          _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));
   
      /* Set the value at the beginning or the end. */
      if (prepend)
          _intsetSet(is,0,value);
      else
          _intsetSet(is,intrev32ifbe(is->length),value);
      is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
      return is;
  }
   
  /* Resize the intset */
  //解除is的内存分配并重新分配长度为len的intset的内存
  static intset *intsetResize(intset *is, uint32_t len) {
      uint32_t size = len*intrev32ifbe(is->encoding);
      is = zrealloc(is,sizeof(intset)+size);
      return is;
  }
   
  //把from索引到intset尾部的整块数据复制to索引(复制之后from值不变，但是可以被覆盖)
  static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
      void *src, *dst;
      uint32_t bytes = intrev32ifbe(is->length)-from;
      uint32_t encoding = intrev32ifbe(is->encoding);
   
      if (encoding == INTSET_ENC_INT64) {
          src = (int64_t*)is->contents+from;
          dst = (int64_t*)is->contents+to;
          bytes *= sizeof(int64_t);
      } else if (encoding == INTSET_ENC_INT32) {
          src = (int32_t*)is->contents+from;
          dst = (int32_t*)is->contents+to;
          bytes *= sizeof(int32_t);
      } else {
          src = (int16_t*)is->contents+from;
          dst = (int16_t*)is->contents+to;
          bytes *= sizeof(int16_t);
      }
      memmove(dst,src,bytes);
  }
  ```

### 移除成员

不同于插入一个成员，移除一个成员时不会改变intset的encoding，尽管移除这个成员之后所有成员的encoding都小于所在intset的encoding。

```c
/* Delete integer from intset */
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 0;
 
    //valenc不可能大于当前编码，否则value一定不在该intset中
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);
 
        /* We know we can delete */
        if (success) *success = 1;
 
        /* Overwrite value with tail and update length */
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);
        is = intsetResize(is,len-1);//减小内存分配
        is->length = intrev32ifbe(len-1);//size-1
    }
    return is;
}
```

## 总结

- intSet是Set结构的一种底层实现，其遵循Set的不重复原则，但是其底层是给元素做了从大到小的排序
- 相比dict，intSet所需的内存更小，而查询复杂度，dict为O（1），intSet为O(log(n))，所以intSet只有在存储少量整形数据时，才有优势
- intSet的实现是一个数组contents，类型是int_8，元素查找是通过二分法查找
- 虽然contents数组的声明是int_8，但其内的元素类型取决于encoding属性，共有int16_t、int32_t 或者int64_t 
- 当新增的元素类型比原集合元素类型的长度要大时（比如：原来是int16_t，现在新增一个int64_t的元素)，需要对intSet升级
- 只提供升级操作，不支持降级操作，**一旦对数组进行了升级，编码就会一直保持升级后的状态。**



## 参考资料

https://blog.csdn.net/zgaoq/article/details/89708500