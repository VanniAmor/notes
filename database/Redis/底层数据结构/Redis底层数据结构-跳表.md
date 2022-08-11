[toc]



【博客】 https://www.cnblogs.com/hunternet/p/11248192.html

 Redis使用跳跃表作为有序集合键的底层实现之一，在了解Redis跳表之前，先看看跳表。

## **基础跳跃表**

对于一个单链表来讲，即便链表中存储的数据是有序的，如果我们要想在其中查找某个数据，也只能从头到尾遍历链表。这样查找效率就会很低，时间复杂度会很高，是 O(n)。

![image-20210119110921516](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119110921516.png)

若想提高其查询效率，可以在链表的基础上创建索引，每两个节点提取到上一级，把抽取出来的索引称为 **一级索引**

![image-20210119111113973](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119111113973.png)

建立索引后，假设要查找节点8，此时可以先在索引层遍历，当遍历到节点7时，发现下一个节点是9，那么8可能在这两个节点之间。我们下降到链表层继续遍历就找到节点8。

**原来单链表的方式查找节点8我们需要遍历8次，而现在有了一级索引只需要遍历5次。**

由此可以看出，**添加索引后，链表的查询效率提高了**，同理继续添加索引

![image-20210119111643286](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119111643286.png)



再次添加了二级索引后，查询节点8只需要遍历4次，分别是：1 -> 5 -> 7 -> 8

从上面例子中可以看出，当有大量数据的时候，可以增加多级索引，其查询效率会有明显提升。

![image-20210119112422998](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119112422998.png)

**像这种链表加多级索引的结构，就是跳跃表**

## **Redis跳跃表**

​	Redis采用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的**元素数量比较多**，或者有序集合中的 **成员是比较长的字符串** 时，Redis就会采用跳跃表来作为有序集合的底层实现。

​	此时我们需要思考的是，为什么元素数量较多 或者 成员是比较长的字符串时， Redis采用跳跃表来实现？

​	从上面我们可以知道，跳跃表在链表的基础上增加了多级索引以提升查找的效率，但其是一个**空间换时间的方案**，必然会带来一个问题——索引是占内存的。原始链表中存储的有可能是很大的对象，而**索引结点只需要存储关键值值和几个指针**，并不需要存储对象，因此当节点本身比较大或者元素数量比较多的时候，其优势必然会被放大，而缺点则可以忽略。

### **Redis跳表结构**

![image-20210119113214238](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119113214238.png)

Redis跳表由两个重要结构组成：zskiplist 和 zskiplistNode，其中zskiplist结构用于保存跳跃表节点的相关信息（节点数量，表头表尾指针等等），zskiplistNode则保存跳表节点。

**zskiplist包含如下信息：**

- header：**指针**， 指向跳表的表头节点，O(1)
- tail: **指针**，指向跳表的表尾节点， O(1)
- level: **int**, 记录目前跳跃表内，表内节点最大层数（表头节点的层数不计算在内）
- length: **unsigned long**， 表内节点数量

**zskiplistNode包含如下信息**

```c
typedef struct zskiplistNode {

    // member 对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 这个层跨越的节点数量(包含自己在内)
        unsigned int span;

    } level[];

} zskiplistNode;
```

- obj（成员对象），即上图中 o1, o2, o3,是用来存储一个节点中的对象的。
- score（分值）， 是成员对象的分值，用于排序
- 后退指针，这个指针指向的是前面的一个跳表节点
- 层，这个结构包括前进指针和记录了跨越的节点数量，这个结构是跳表的精髓（每个节点最多持有32个）



## **Redis跳表的增删改查**



### **查找过程**

![image-20210119170745911](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119170745911.png)

如上图，假设要查找节点23

- 从最高层开始尝试移动，比较next节点的分值和待查找分值大小
- 如果next节点的分值小于待查找分值，则移动到next节点
- 如果next节点的分值大于等于待查找分值，则降层，如果此时为第0层不能继续降层，当前节点位置就是待查找节点的前一个节点
- 查找完成后，找到的节点的下一个节点不是待查找节点，就是分值23应该插入的位置

查找时间复杂度为 O(log(n))

### **插入过程**

https://blog.csdn.net/u013536232/article/details/105476382/

![image-20210119150015825](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119150015825.png)

假设要插入9.5这个节点

总共有以下几个步骤

- 找到新节点在每一层上的每一个节点（对于level0是节点9，对于level1是节点8）
- 将新节点插入到每一层的上一个节点和下一个节点之间 
- 重新计算span（某节点到下一节点跨越的节点数）
- 随机生成插入层数i
- 更新整个跳表的最高层数

#### **遍历，记录update和rank**

首先创建两个数组，数组的大小都是最大层数，即32

- **update数组**用来记录新节点在每一层的上一个节点，也就是新节点要插到哪个节点后面；
- **rank数组**用来记录update节点的排名，也就是在这一层，update节点到头节点的距离，这个上一节说过，是为了用来计算span。

**update和rank数组的值可以通过一次逐层的遍历确定。**

在遍历之前，定义一个x，指向头节点

![image-20210119162512283](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119162512283.png)

遍历过程可以参照上图，还是假设插入节点9.5。遍历从最高层开始，一直到最底层

遍历过程中，若x的forward指针指向的节点（即x的下一个节点）的score低于插入节点的评分，那么插入节点应该在x的下一个节点的右侧。所以此时rank应该加上x节点的span（即x到x下一节点的距离），然后x指向x的下一个节点，继续循环。

- 第一次遍历，就是level2，找到的update节点是节点8，对应的rank是此时x节点的span，也就是头节点在level2的span。x这时也移动到了节点8。

- 第二次遍历，注意这里不是再重头开始遍历了，因为上一次遍历完，x已经移到了level2的update节点，而在level1，update节点一定在x当前的位置之后（>=），所以对于rank的计算，也可以直接在上一层的rank的基础上继续计算。

完整代码如下：

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;
 
    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    /* we assume the element is not already inside, since we allow duplicated
     * scores, reinserting the same element should never happen since the
     * caller of zslInsert() should test in the hash table if the element is
     * already inside or not. */
    level = zslRandomLevel();
    if (level > zsl->level) {
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        zsl->level = level;
    }
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;
 
        /* update span covered by update[i] as x is inserted here */
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }
 
    /* increment span for untouched levels */
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }
 
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;
    return x;
}
```



#### 生成随机层数

```c
// https://zhuanlan.zhihu.com/p/92536201
// ZSKIPLIST_P = 0.25
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

这个函数符合幂次定律

![image-20210119151717026](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119151717026.png)

**幂次定律应用到跳表的随机层数来说就是大部分的节点层数都是黄色部分，只有少数是绿色部分，并且概率很低。**

### 删除节点

删除节点需要先找到节点的位置，如果找到，则将其删除。删除节点和插入节点一样，都会破坏某些节点的next和span，所以需要对结构进行更新



```c
//t_zset.c
/* 从跳表中删除节点 */
void zslDeleteNode(zskiplist *zsl, zskiplistNode *x, zskiplistNode **update) {
    int i;
    /* 对于每一层，改变其next数组和跨度 */
    for (i = 0; i < zsl->level; i++) {
        /* 如果当前节点的当前层的next节点是要删除的节点，改变其next指针和跨度 */
        if (update[i]->level[i].forward == x) {
            update[i]->level[i].span += x->level[i].span - 1;
            update[i]->level[i].forward = x->level[i].forward;
        } else {
            /* 否则，只需要改变跨度 */
            update[i]->level[i].span -= 1;
        }
    }
    /* 删除节点后面的后继节点的前驱指针也需要改变 */
    if (x->level[0].forward) {
        x->level[0].forward->backward = x->backward;
    } else {
        zsl->tail = x->backward;
    }
    /* 如果删除的是最高层节点，同时删除后最高层为空，就将跳表层数降低 */
    while(zsl->level > 1 && zsl->header->level[zsl->level-1].forward == NULL)
        zsl->level--;
    /* 节点个数减少 */
    zsl->length--;
}

/* 删除分值score和数据obj匹配的节点 */
int zslDelete(zskiplist *zsl, double score, robj *obj) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    /* 寻找待删除节点的前一个节点，和插入操作的查找相同 */
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                compareStringObjects(x->level[i].forward->obj,obj) < 0)))
            x = x->level[i].forward;
        update[i] = x;
    }

    /* 判断是否存在要删除的节点，x是插入位置前的节点，那么它的next指针就是需要删除的节点 */
    x = x->level[0].forward;
    if (x && score == x->score && equalStringObjects(x->obj,obj)) {
        /* 如果是，则调用删除节点操作 */
        zslDeleteNode(zsl, x, update);
        zslFreeNode(x);
        return 1;
    }
    return 0; /* not found */
}
```



### 节点更新

更新和插入，其实都是使用zadd方法，先判断这个value是否存在，如果存在则执行更新操作，若不存在，则执行插入操作。插入操作即上面的`zslInsert`方法

而更新操作，过程是：先遍历查找value，若找到，则删除后，再重新插入。这样的弊端是会做两次遍历。

在Redis5.0版本中，作者修改了这个过程，并新增了一个更新方法，如下

```c
/**
 * 先判断这个value是否存在，若存在则直接更新，然后调整跳表的score排序
 */
zskiplistNode *zslUpdateScore(zskiplist *zsl, double curscore, sds ele, double newscore) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    int i;

    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        while (x->level[i].forward &&
                (x->level[i].forward->score < curscore ||
                    (x->level[i].forward->score == curscore &&
                     sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            x = x->level[i].forward;
        }
        update[i] = x;
    }

    /* Jump to our element: note that this function assumes that the
     * element with the matching score exists. */
    x = x->level[0].forward;
    serverAssert(x && curscore == x->score && sdscmp(x->ele,ele) == 0);

    /* If the node, after the score update, would be still exactly
     * at the same position, we can just update the score without
     * actually removing and re-inserting the element in the skiplist. */
    if ((x->backward == NULL || x->backward->score <= newscore) &&
        (x->level[0].forward == NULL || x->level[0].forward->score >= newscore))
    {
        x->score = newscore;
        return x;
    }

    /* No way to reuse the old node: we need to remove and insert a new
     * one at a different place. */
    zslDeleteNode(zsl, x, update);
    zskiplistNode *newnode = zslInsert(zsl,newscore,x->ele);
    /* We reused the old node x->ele SDS string, free the node now
     * since zslInsert created a new one. */
    x->ele = NULL;
    zslFreeNode(x);
    return newnode;
}

```

### **跳表的实际使用场景**

![image-20210119172514709](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20210119172514709.png)

Zset 的数据结构是有一个 hash 表和一个跳跃表来结合的，hash 表上存储的是关于 String 的值和 Score 的值，跳跃表是用来辅助 hash 表来实现关于按照 score 来排序的功能。

- 在 Zset 中使用最多的场景就是涉及到排行榜类似的场景。例如实时统计一个关于分数的排行榜，这个时候可以使用 Redis 中的这个 ZSET 数据结构来维护。
- 涉及到需要按照时间的顺序来排行的业务场景，例如如果需要维护一个问题池，按照时间的先后顺序来维护，这个时候也可以使用 Zset ，把时间当做权重，把问题当做 key 值来进行存取。

- 延迟队列，把任务id和延迟时间放到zset中，开一个线程轮询，弹出需要执行的任务id

...

### **跳表与红黑树**

有序元素的查找，除了跳表，更为常见的是红黑树为代表的一众平衡树结构。那么为何Redis作者选择skiptable呢

- 平衡树的插入 和 删除操作可能引发树的旋转调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。（跳表实现简单）

- 在并发的情况下，红黑树在插入删除的时候可能需要做rebalance的操作，这样的操作可能会涉及到整个树的其他部分；而链表的操作就会相对局部，只需要关注插入删除的位置即可，只要多个线程操作的地方不一样，就不会产生冲突

ps：哈希表是无序的，只适合做单个key的查询，不适合做范围查找