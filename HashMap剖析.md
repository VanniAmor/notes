# HashMap剖析

    参考文章
    
        https://segmentfault.com/a/1190000021928659
        https://my.oschina.net/u/2307589/blog/1800587
        https://blog.csdn.net/v123411739/article/details/106324537
        https://coolshell.cn/articles/9606.html
        https://blog.csdn.net/v123411739/article/details/78996181
        https://blog.csdn.net/visant/article/details/80045154
        https://www.iteye.com/topic/539465
        https://www.jianshu.com/p/e136ec79235c
        https://zhuanlan.zhihu.com/p/79980618
    
### 代码继承关系

[![tqKJxS.png](https://s1.ax1x.com/2020/06/11/tqKJxS.png)](https://imgchr.com/i/tqKJxS)

- HashMap根据键的hashcode值存储数据，大多数情况下可以直接定位到key的值，因此有很快的访问速度，但遍历顺序是不确定的。
- **HashMap最多只允许一条记录的键为null，允许多条记录的值为null**。
- HashMap非线程安全，多个线程同时操作，会出现HashMap死循环，CPU占用率飙升，这个问题稍后详细说。**多线程操作应该使用ConcurrentHashMap**


### HashMap的数据结构
HashMap在JDK1.7前后有很大的变化，JDK1.7及以前，底层实现是“数组 + 链表”，JDK1.8后改为“数组 + 链表 + 红黑树”

1.8的HashMap数据结构如下图所示，HashMap中使用链表和红黑树来解决哈希碰撞问题，当链表长度大于8时，转为使用红黑树来解决哈希冲突

[![tquAXQ.png](https://s1.ax1x.com/2020/06/11/tquAXQ.png)](https://imgchr.com/i/tquAXQ)

当往HashMap中put元素的时候，先根据key的hash值得到这个元素在数组中的位置（即下标），若同一位置上已经存放其他元素了，那么将以链表的形式（或红黑树）存放，新插入的元素放在链尾。当需要get元素时，需要计算key的hashcode，然后通过hash算法获取到该key在数组的下标。可以想象，当数组中每个位置上都只有一个元素，不用遍历链表/红黑树，那么hashmap的效率将是最高的。那么为了提高hashMap的检索效率，hashMap的长度和hash算法的设计就很有学问了。

### 定位哈希桶数组索引位置

计算key的下标，首先想到的是把hashcode对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是“模运算”消耗是比较大的，**java中采用与操作替代模运算，但计算结果是一样的**

```
// 代码1
static final int hash(Object key) { // 计算key的hash值
    int h;
    // 1.先拿到key的hashCode值; 2.将hashCode的高16位参与运算
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 代码2
// 将计算得到的hashcode与tabel.length - 1进行 与操作
static int indexFor(int h, int length) {  
   return h & (length-1);  
}  
```
    上述优化是基于此条公式
    x mod 2^n = x & (2^n - 1)
    
    整个过程的本质就是
    1. 拿到key的hashcode
    2. 将hashcode的高位参与位运算，重新计算hash值
    3. 将计算到的hash值与(table.length-1)进行 & 操作

这里有个重要的前提，就是**HashMap长度一定是2的n次幂**，下面将以2^4作讲解

假设计算到的hashcode为8和9，而两个hashmap的长度，一个是16，一个是15，以此计算这两个hashcode的下标

[![tLrirQ.jpg](https://s1.ax1x.com/2020/06/12/tLrirQ.jpg)](https://imgchr.com/i/tLrirQ)

可以看到，当hashmap长度为15时，hashcode会与14（1110）进行与操作，那么最后一位永远都是0，而0001，0011，0101，1001，1011，0111，1101这几个位置永远都不能存放元素了。这样一来，数组中可以使用的位置就比数组长度小了很多，这意味着增加了哈希碰撞的几率

**要想hashcode的碰撞减到最少，就要跟一个 1...1的数进行与操作，即2^n -1,所以hashmap的长度设置为2的n次幂是最为合理的**

而且上面代码中计算key的hash值时，是将hashcode的高16位与hashcode进行异或运算，主要是为了在table的长度较少时，让高位也参与运算，且不会有太大的计算开销

![tL5SG4.png](https://s1.ax1x.com/2020/06/12/tL5SG4.png)

【总结】

1. 位运算操作效率最高
2. hashcode逻辑右移16位参与hash值计算，是为了在table长度较少时也参与计算
3. table的长度设置为2的n次幂，是为了减少hash碰撞的记录，增加hashmap的检索几率
    
---

# HashMap源码解析（JDK1.8）

- HashMap本质是一个Node<K,V>[] 数组
- 链表节点Node<K,V>通过next属性维护链表
- 转为红黑树后，链表结构仍然存在，红黑树进行操作时仍会维护链表，当转为红黑树后，树的节点被删除到少于6个后，会重新变为链表结构
- 红黑树的遍历时，使用变量dir（direction）表示遍历的方向，dir 存储的值是目标节点的 hash/key 与 p 节点的 hash/key 的比较结果，-1时向左遍历，1时向右遍历
- 链表转化红黑树时，是先将链表节点转化为红黑树节点，然后再构建红黑树，红黑树节点插入也是这样。
- 

## HashMap常量

```
// 默认容量16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 
 
// 最大容量
static final int MAXIMUM_CAPACITY = 1 << 30;    
 
// 默认负载因子0.75
static final float DEFAULT_LOAD_FACTOR = 0.75f; 
 
// 链表节点转换红黑树节点的阈值, 9个节点转
static final int TREEIFY_THRESHOLD = 8; 
 
// 红黑树节点转换链表节点的阈值, 6个节点转
static final int UNTREEIFY_THRESHOLD = 6;   
 
// 转红黑树时, table的最小长度
static final int MIN_TREEIFY_CAPACITY = 64; 
```

## HashMap Fields
```
/**
 * The table, initialized on first use, and resized as
 * necessary. When allocated, length is always a power of two.
 * (We also tolerate length zero in some operations to allow
 * bootstrapping mechanics that are currently not needed.)
 * 保存Node的数组，即不发生哈希冲突的情况下，Node将会保存在这个数组
 */
transient Node<K,V>[] table;

/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 * 由　HashMap 中 Node<K,V>　节点构成的 set
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * The number of key-value mappings contained in this map.
 * 记录HashMap中当前存储的元素的数量
 */
transient int size;

/**
 * The number of times this HashMap has been structurally modified
 * Structural modifications are those that change the number of mappings in
 * the HashMap or otherwise modify its internal structure (e.g.,
 * rehash).  This field is used to make iterators on Collection-views of
 * the HashMap fail-fast.  (See ConcurrentModificationException).
 * 记录　HashMap 发生结构性变化的次数（注意　value 的覆盖不属于结构性变化）
 */
transient int modCount;

/**
 * The next size value at which to resize (capacity * load factor).
 *
 * @serial
 */
// (The javadoc description is true upon serialization.
// Additionally, if the table array has not been allocated, this
// field holds the initial array capacity, or zero signifying
// DEFAULT_INITIAL_CAPACITY.)
// 拓容阀值，threshold 等于 table.length * loadFactor，size 超过这个值时进行　resize()扩容
int threshold;

/**
 * The load factor for the hash table.
 *
 * @serial
 * 记录负载因子
 */
final float loadFactor;
```

## HashMap 链表节点
```
// 链表节点, 继承自Entry
static class Node<K,V> implements Map.Entry<K,V> {  
    final int hash;
    final K key;
    V value;
    Node<K,V> next;
 
    // ... ...
}
```

## HashMap 红黑树节点

```
// 红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
   
    // ...
}
```

## HashMap初始化

四种构造方法
```
/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 * 无参构造函数，所有字段都是用默认值
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and the default load factor (0.75).
 * 指定初始容量，使用默认负载因子
 *
 * @param  initialCapacity the initial capacity.
 * @throws IllegalArgumentException if the initial capacity is negative.
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * Constructs an empty <tt>HashMap</tt> with the specified initial
 * capacity and load factor.
 * 指定初始容量和负载因子
 *
 * @param  initialCapacity the initial capacity
 * @param  loadFactor      the load factor
 * @throws IllegalArgumentException if the initial capacity is negative
 *         or the load factor is nonpositive
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

 /**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 * 传入Map作为参数，使用默认负载因子
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

```


## HashMap插入


1. 先检测HashMap是否为空，若为空，则调用resize方法初始化HashMap
2. 计算插入key的下标, 通过Key 的 hashCode & (length-1)来计算
3. 如果table[i] == null, 直接插入
4. 若table[i]有值，则判断是否为红黑树节点TreeNode<K,V>,是的话则执行红黑树节点的插入
5. 判断是否链表节点Node<K,V>,则进行链表节点的插入,插入后再判断是否需要转为红黑树

#### putVal方法

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 1.校验table是否为空或者length等于0，如果是则调用resize方法进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2.通过hash值计算索引位置，将该索引位置的头节点赋值给p，如果p为空则直接在该索引位置新增一个节点即可
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // table表该索引位置不为空，则进行查找
        Node<K,V> e; K k;
        // 3.判断p节点的key和hash值是否跟传入的相等，如果相等, 则p节点即为要查找的目标节点，将p节点赋值给e节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 4.判断p节点是否为TreeNode, 如果是则调用红黑树的putTreeVal方法查找目标节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 5.走到这代表p节点为普通链表节点，则调用普通的链表方法进行查找，使用binCount统计链表的节点数
            for (int binCount = 0; ; ++binCount) {
                // 6.如果p的next节点为空时，则代表找不到目标节点，则新增一个节点并插入链表尾部
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 7.校验节点数是否超过8个，如果超过则调用treeifyBin方法将链表节点转为红黑树节点，
                    // 减一是因为循环是从p节点的下一个节点开始的
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 8.如果e节点存在hash值和key值都与传入的相同，则e节点即为目标节点，跳出循环
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;  // 将p指向下一个节点
            }
        }
        // 9.如果e节点不为空，则代表目标节点存在，使用传入的value覆盖该节点的value，并返回oldValue
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e); // 用于LinkedHashMap
            return oldValue;
        }
    }
    ++modCount;
    // 10.如果插入节点后节点数超过阈值，则调用resize方法进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);  // 用于LinkedHashMap
    return null;
}
```
### putTreeVal方法，插入红黑树节点

- 红黑树的遍历通过两条规则
    
```
1. 若目标节点的hash值小于当前节点的hash，则向当前节点的左边遍历
2. 若目标节点的hash值小于当前节点的hash，则向当前节点的右边遍历

至于hash值大小的比较，若两者相同时，则查看当前key是否有实现Compable接口，若有，则使用定义好的比较器进行比较
若无，则使用tieBreakOrder进行比较，这是极端情况下的比较
```

- 当通过比较找到需要插入的位置X后，新建TreeNode节点XN，且将父节点XP的next属性设置为XN，这是维护链表的操作


```
/**
 * 红黑树的put操作，红黑树插入会同时维护原来的链表属性, 即原来的next属性
 */
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                               int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false;
    // 1.查找根节点, 索引位置的头节点并不一定为红黑树的根节点
    TreeNode<K,V> root = (parent != null) ? root() : this;
    // 2.将根节点赋值给p节点，开始进行查找
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        // 3.如果传入的hash值小于p节点的hash值，将dir赋值为-1，代表向p的左边查找树
        if ((ph = p.hash) > h)
            dir = -1;
        // 4.如果传入的hash值大于p节点的hash值， 将dir赋值为1，代表向p的右边查找树
        else if (ph < h)
            dir = 1;
        // 5.如果传入的hash值和key值等于p节点的hash值和key值, 则p节点即为目标节点, 返回p节点
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        // 6.如果k所属的类没有实现Comparable接口 或者 k和p节点的key相等
        else if ((kc == null &&
                  (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            // 6.1 第一次符合条件, 从p节点的左节点和右节点分别调用find方法进行查找, 如果查找到目标节点则返回
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null &&
                     (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null &&
                     (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            // 6.2 否则使用定义的一套规则来比较k和p节点的key的大小, 用来决定向左还是向右查找
            dir = tieBreakOrder(k, pk); // dir<0则代表k<pk，则向p左边查找；反之亦然
        }
 
        TreeNode<K,V> xp = p;   // xp赋值为x的父节点,中间变量,用于下面给x的父节点赋值
        // 7.dir<=0则向p左边查找,否则向p右边查找,如果为null,则代表该位置即为x的目标位置
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            // 走进来代表已经找到x的位置，只需将x放到该位置即可
            Node<K,V> xpn = xp.next;    // xp的next节点
            // 8.创建新的节点, 其中x的next节点为xpn, 即将x节点插入xp与xpn之间
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            // 9.调整x、xp、xpn之间的属性关系
            if (dir <= 0)   // 如果时dir <= 0, 则代表x节点为xp的左节点
                xp.left = x;
            else        // 如果时dir> 0, 则代表x节点为xp的右节点
                xp.right = x;
            xp.next = x;    // 将xp的next节点设置为x
            x.parent = x.prev = xp; // 将x的parent和prev节点设置为xp
            // 如果xpn不为空,则将xpn的prev节点设置为x节点,与上文的x节点的next节点对应
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            // 10.进行红黑树的插入平衡调整
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}
```
**当节点插入链表，链表节点数大8，进行链表 => 红黑树的转化，有两个步骤**

**1. 执行treeifyBin方法，将Node节点转化为TreeNode节点**

**2. 执行treeify方法，构建红黑树**

### treeifyBin方法
```
/**
 * 将链表节点转为红黑树节点
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 1.如果table为空或者table的长度小于64, 调用resize方法进行扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    // 2.根据hash值计算索引值，将该索引位置的节点赋值给e，从e开始遍历该索引位置的链表
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 3.将链表节点转红黑树节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // 4.如果是第一次遍历，将头节点赋值给hd
            if (tl == null)	// tl为空代表为第一次循环
                hd = p;
            else {
                // 5.如果不是第一次遍历，则处理当前节点的prev属性和上一个节点的next属性
                p.prev = tl;    // 当前节点的prev属性设为上一个节点
                tl.next = p;    // 上一个节点的next属性设置为当前节点
            }
            // 6.将p节点赋值给tl，用于在下一次循环中作为上一个节点进行一些链表的关联操作（p.prev = tl 和 tl.next = p）
            tl = p;
        } while ((e = e.next) != null);
        // 7.将table该索引位置赋值为新转的TreeNode的头节点，如果该节点不为空，则以以头节点(hd)为根节点, 构建红黑树
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

### treeify方法

遍历TreeNode数组，通过比较数组中元素key的hash值，小的在左子树，大的在右子树，构建红黑树

```
/**
 * 构建红黑树
 */
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    // 1.将调用此方法的节点赋值给x，以x作为起点，开始进行遍历
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;   // next赋值为x的下个节点
        x.left = x.right = null;    // 将x的左右节点设置为空
        // 2.如果还没有根节点, 则将x设置为根节点
        if (root == null) {
            x.parent = null;    // 根节点没有父节点
            x.red = false;  // 根节点必须为黑色
            root = x;   // 将x设置为根节点
        }
        else {
            K k = x.key;	// k赋值为x的key
            int h = x.hash;	// h赋值为x的hash值
            Class<?> kc = null;
            // 3.如果当前节点x不是根节点, 则从根节点开始查找属于该节点的位置
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                // 4.如果x节点的hash值小于p节点的hash值，则将dir赋值为-1, 代表向p的左边查找
                if ((ph = p.hash) > h)
                    dir = -1;
                // 5.如果x节点的hash值大于p节点的hash值，则将dir赋值为1, 代表向p的右边查找
                else if (ph < h)
                    dir = 1;
                // 6.走到这代表x的hash值和p的hash值相等，则比较key值
                else if ((kc == null && // 6.1 如果k没有实现Comparable接口 或者 x节点的key和p节点的key相等
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    // 6.2 使用定义的一套规则来比较x节点和p节点的大小，用来决定向左还是向右查找
                    dir = tieBreakOrder(k, pk);
 
                TreeNode<K,V> xp = p;   // xp赋值为x的父节点,中间变量用于下面给x的父节点赋值
                // 7.dir<=0则向p左边查找,否则向p右边查找,如果为null,则代表该位置即为x的目标位置
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    // 8.x和xp节点的属性设置
                    x.parent = xp;  // x的父节点即为最后一次遍历的p节点
                    if (dir <= 0)   // 如果时dir <= 0, 则代表x节点为父节点的左节点
                        xp.left = x;
                    else    // 如果时dir > 0, 则代表x节点为父节点的右节点
                        xp.right = x;
                    // 9.进行红黑树的插入平衡(通过左旋、右旋和改变节点颜色来保证当前树符合红黑树的要求)
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    // 10.如果root节点不在table索引位置的头节点, 则将其调整为头节点
    moveRootToFront(tab, root);
}
```


## HashMap获取

1. 计算key的hashcode，并用此hashcode计算在HashMap数组的下标i
2. 获取HashMap中table[i]的节点first
3. 判断节点first的key是否与入参的key相同，若相同，则返回first
4. first与入参key不同，则遍历链表/红黑树，通过比较节点的hash和入参key的hash，直到找到对应的节点
5. 都找不到，返回null

### get方法（获取节点）

```
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
 
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 1.对table进行校验：table不为空 && table长度大于0 && 
    // table索引位置(使用table.length - 1和hash值进行位与运算)的节点不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 2.检查first节点的hash值和key是否和入参的一样，如果一样则first即为目标节点，直接返回first节点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 3.如果first不是目标节点，并且first的next节点不为空则继续遍历
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                // 4.如果是红黑树节点，则调用红黑树的查找目标节点方法getTreeNode
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                // 5.执行链表节点的查找，向下遍历链表, 直至找到节点的key和入参的key相等时,返回该节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    // 6.找不到符合的返回空
    return null;
}
```

### getTreeNode，查找红黑树中的节点

```
// 以下方法皆在TreeNode<K,V>内部类中
final TreeNode<K,V> getTreeNode(int h, Object k) {
    // 1.首先找到红黑树的根节点；2.使用根节点调用find方法
    return ((parent != null) ? root() : this).find(h, k, null);
}

/**
 * 从调用此方法的节点开始查找, 通过hash值和key找到对应的节点
 * 此方法是红黑树节点的查找, 红黑树是特殊的自平衡二叉查找树
 * 平衡二叉查找树的特点：左节点<根节点<右节点
 */
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    // 1.将p节点赋值为调用此方法的节点，即为红黑树根节点
    TreeNode<K,V> p = this;
    // 2.从p节点开始向下遍历
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        // 3.如果传入的hash值小于p节点的hash值，则往p节点的左边遍历
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h) // 4.如果传入的hash值大于p节点的hash值，则往p节点的右边遍历
            p = pr;
        // 5.如果传入的hash值和key值等于p节点的hash值和key值,则p节点为目标节点,返回p节点
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)    // 6.p节点的左节点为空则将向右遍历
            p = pr;
        else if (pr == null)    // 7.p节点的右节点为空则向左遍历
            p = pl;
        // 8.将p节点与k进行比较
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) && // 8.1 kc不为空代表k实现了Comparable
                 (dir = compareComparables(kc, k, pk)) != 0)// 8.2 k<pk则dir<0, k>pk则dir>0
            // 8.3 k<pk则向左遍历(p赋值为p的左节点), 否则向右遍历
            p = (dir < 0) ? pl : pr;
        // 9.代码走到此处, 代表key所属类没有实现Comparable, 直接指定向p的右边遍历
        else if ((q = pr.find(h, k, kc)) != null) 
            return q;
        // 10.代码走到此处代表“pr.find(h, k, kc)”为空, 因此直接向左遍历
        else
            p = pl;
    } while (p != null);
    return null;
}
```

## HashMap节点删除

1. 根据入参key，通过HashMap的查找，找到需要删除的节点（Node节点或TreeNode节点）
2. 若是Note节点，进行链表节点删除
3. 若是TreeNode节点，进行红黑树节点删除
4. 将删除的节点返回

### remove方法

```
/**
 * 移除某个节点
 */
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
 
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 1.如果table不为空并且根据hash值计算出来的索引位置不为空, 将该位置的节点赋值给p
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 2.如果p的hash值和key都与入参的相同, 则p即为目标节点, 赋值给node
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 3.否则将p.next赋值给e，向下遍历节点
            // 3.1 如果p是TreeNode则调用红黑树的方法查找节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 3.2 否则，进行普通链表节点的查找
                do {
                    // 当节点的hash值和key与传入的相同,则该节点即为目标节点
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;	// 赋值给node, 并跳出循环
                        break;
                    }
                    p = e;  // p节点赋值为本次结束的e，在下一次循环中，e为p的next节点
                } while ((e = e.next) != null); // e指向下一个节点
            }
        }
        // 4.如果node不为空(即根据传入key和hash值查找到目标节点)，则进行移除操作
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            // 4.1 如果是TreeNode则调用红黑树的移除方法
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 4.2 如果node是该索引位置的头节点则直接将该索引位置的值赋值为node的next节点，
            // “node == p”只会出现在node是头节点的时候，如果node不是头节点，则node为p的next节点
            else if (node == p)
                tab[index] = node.next;
            // 4.3 否则将node的上一个节点的next属性设置为node的next节点,
            // 即将node节点移除, 将node的上下节点进行关联(链表的移除)
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node); // 供LinkedHashMap使用
            // 5.返回被移除的节点
            return node;
        }
    }
    return null;
}
```

### 红黑树节点删除

```
/**
 * 红黑树的节点移除
 */
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                          boolean movable) {
    // --- 链表的处理start ---
    int n;
    // 1.table为空或者length为0直接返回
    if (tab == null || (n = tab.length) == 0)
        return;
    // 2.根据hash计算出索引的位置
    int index = (n - 1) & hash;
    // 3.将索引位置的头节点赋值给first和root
    TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
    // 4.该方法被将要被移除的node(TreeNode)调用, 因此此方法的this为要被移除node节点,
    // 将node的next节点赋值给succ节点，prev节点赋值给pred节点
    TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
    // 5.如果pred节点为空，则代表要被移除的node节点为头节点，
    // 则将table索引位置的值和first节点的值赋值为succ节点(node的next节点)即可
    if (pred == null)
        tab[index] = first = succ;
    else
        // 6.否则将pred节点的next属性设置为succ节点(node的next节点)
        pred.next = succ;
    // 7.如果succ节点不为空，则将succ的prev节点设置为pred, 与前面对应
    if (succ != null)
        succ.prev = pred;
    // 8.如果进行到此first节点为空，则代表该索引位置已经没有节点则直接返回
    if (first == null)
        return;
    // 9.如果root的父节点不为空, 则将root赋值为根节点
    if (root.parent != null)
        root = root.root();
    // 10.通过root节点来判断此红黑树是否太小, 如果是则调用untreeify方法转为链表节点并返回
    // (转链表后就无需再进行下面的红黑树处理)
    if (root == null || root.right == null ||
        (rl = root.left) == null || rl.left == null) {
        tab[index] = first.untreeify(map);  // too small
        return;
    }
    // --- 链表的处理end ---
 
    // --- 以下代码为红黑树的处理 ---
    // 11.将p赋值为要被移除的node节点，pl赋值为p的左节点，pr赋值为p 的右节点
    TreeNode<K,V> p = this, pl = left, pr = right, replacement;
    // 12.如果p的左节点和右节点都不为空时
    if (pl != null && pr != null) {
        // 12.1 将s节点赋值为p的右节点
        TreeNode<K,V> s = pr, sl;
        // 12.2 向左一直查找，跳出循环时,s为没有左节点的节点
        while ((sl = s.left) != null)
            s = sl;
        // 12.3 交换p节点和s节点的颜色
        boolean c = s.red; s.red = p.red; p.red = c;
        TreeNode<K,V> sr = s.right; // s的右节点
        TreeNode<K,V> pp = p.parent;    // p的父节点
        // --- 第一次调整和第二次调整：将p节点和s节点进行了位置调换 ---
        // 12.4 第一次调整
        // 如果p节点的右节点即为s节点，则将p的父节点赋值为s，将s的右节点赋值为p
        if (s == pr) {
            p.parent = s;
            s.right = p;
        }
        else {
            // 将sp赋值为s的父节点
            TreeNode<K,V> sp = s.parent;
            // 将p的父节点赋值为sp
            if ((p.parent = sp) != null) {
                // 如果s节点为sp的左节点，则将sp的左节点赋值为p节点
                if (s == sp.left)
                    sp.left = p;
                // 否则s节点为sp的右节点，则将sp的右节点赋值为p节点
                else
                    sp.right = p;
            }
            // s的右节点赋值为p节点的右节点
            if ((s.right = pr) != null)
                // 如果pr不为空，则将pr的父节点赋值为s
                pr.parent = s;
        }
        // 12.5 第二次调整
        // 将p的左节点赋值为空，pl已经保存了该节点
        p.left = null;
        // 将p节点的右节点赋值为sr，如果sr不为空，则将sr的父节点赋值为p节点
        if ((p.right = sr) != null)
            sr.parent = p;
        // 将s节点的左节点赋值为pl，如果pl不为空，则将pl的父节点赋值为s节点
        if ((s.left = pl) != null)
            pl.parent = s;
        // 将s的父节点赋值为p的父节点pp
        // 如果pp为空，则p节点为root节点, 交换后s成为新的root节点
        if ((s.parent = pp) == null)
            root = s;
        // 如果p不为root节点, 并且p是pp的左节点，则将pp的左节点赋值为s节点
        else if (p == pp.left)
            pp.left = s;
        // 如果p不为root节点, 并且p是pp的右节点，则将pp的右节点赋值为s节点
        else
            pp.right = s;
        // 12.6 寻找replacement节点，用来替换掉p节点
        // 12.6.1 如果sr不为空，则replacement节点为sr，因为s没有左节点，所以使用s的右节点来替换p的位置
        if (sr != null)
            replacement = sr;
        // 12.6.1 如果sr为空，则s为叶子节点，replacement为p本身，只需要将p节点直接去除即可
        else
            replacement = p;
    }
    // 13.承接12点的判断，如果p的左节点不为空，右节点为空，replacement节点为p的左节点
    else if (pl != null)
        replacement = pl;
    // 14.如果p的右节点不为空,左节点为空，replacement节点为p的右节点
    else if (pr != null)
        replacement = pr;
    // 15.如果p的左右节点都为空, 即p为叶子节点, replacement节点为p节点本身
    else
        replacement = p;
    // 16.第三次调整：使用replacement节点替换掉p节点的位置，将p节点移除
    if (replacement != p) { // 如果p节点不是叶子节点
        // 16.1 将p节点的父节点赋值给replacement节点的父节点, 同时赋值给pp节点
        TreeNode<K,V> pp = replacement.parent = p.parent;
        // 16.2 如果p没有父节点, 即p为root节点，则将root节点赋值为replacement节点即可
        if (pp == null)
            root = replacement;
        // 16.3 如果p不是root节点, 并且p为pp的左节点，则将pp的左节点赋值为替换节点replacement
        else if (p == pp.left)
            pp.left = replacement;
        // 16.4 如果p不是root节点, 并且p为pp的右节点，则将pp的右节点赋值为替换节点replacement
        else
            pp.right = replacement;
        // 16.5 p节点的位置已经被完整的替换为replacement, 将p节点清空, 以便垃圾收集器回收
        p.left = p.right = p.parent = null;
    }
    // 17.如果p节点不为红色则进行红黑树删除平衡调整
    // (如果删除的节点是红色则不会破坏红黑树的平衡无需调整)
    TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);
 
    // 18.如果p节点为叶子节点, 则简单的将p节点去除即可
    if (replacement == p) {
        TreeNode<K,V> pp = p.parent;
        // 18.1 将p的parent属性设置为空
        p.parent = null;
        if (pp != null) {
            // 18.2 如果p节点为父节点的左节点，则将父节点的左节点赋值为空
            if (p == pp.left)
                pp.left = null;
            // 18.3 如果p节点为父节点的右节点， 则将父节点的右节点赋值为空
            else if (p == pp.right)
                pp.right = null;
        }
    }
    if (movable)
        // 19.将root节点移到索引位置的头节点
        moveRootToFront(tab, r);
}
```

# HashMap拓容

两种情况触发resize方法

1. put方法执行时，Node<K,V>[] table 数组仍为空
2. 当table数组容量大于thresold时（capacity * loadFactory）
3. 每次拓容后，新容量是旧容量的两倍
4. 拓容后若老表不为空，需要重新计算节点的位置，这里用到rehash算法

### resize方法

```
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 1.老表的容量不为0，即老表不为空
    if (oldCap > 0) {
        // 1.1 判断老表的容量是否超过最大容量值：如果超过则将阈值设置为Integer.MAX_VALUE，并直接返回老表,
        // 此时oldCap * 2比Integer.MAX_VALUE大，因此无法进行重新分布，只是单纯的将阈值扩容到最大
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 1.2 将newCap赋值为oldCap的2倍，如果newCap<最大容量并且oldCap>=16, 则将新阈值设置为原来的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 2.如果老表的容量为0, 老表的阈值大于0, 是因为初始容量被放入阈值，则将新表的容量设置为老表的阈值
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        // 3.老表的容量为0, 老表的阈值为0，这种情况是没有传初始容量的new方法创建的空表，将阈值和容量设置为默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 4.如果新表的阈值为空, 则通过新的容量*负载因子获得阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 5.将当前阈值设置为刚计算出来的新的阈值，定义新表，容量为刚计算出来的新容量，将table设置为新定义的表。
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 6.如果老表不为空，则需遍历所有节点，将节点赋值给新表
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {  // 将索引值为j的老表头节点赋值给e
                oldTab[j] = null; // 将老表的节点设置为空, 以便垃圾收集器回收空间
                // 7.如果e.next为空, 则代表老表的该位置只有1个节点，计算新表的索引位置, 直接将该节点放在该位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 8.如果是红黑树节点，则进行红黑树的重hash分布(跟链表的hash分布基本相同)
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 9.如果是普通的链表节点，则进行普通的重hash分布
                    Node<K,V> loHead = null, loTail = null; // 存储索引位置为:“原索引位置”的节点
                    Node<K,V> hiHead = null, hiTail = null; // 存储索引位置为:“原索引位置+oldCap”的节点
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 9.1 如果e的hash值与老表的容量进行与运算为0,则扩容后的索引位置跟老表的索引位置一样
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null) // 如果loTail为空, 代表该节点为第一个节点
                                loHead = e; // 则将loHead赋值为第一个节点
                            else
                                loTail.next = e;    // 否则将节点添加在loTail后面
                            loTail = e; // 并将loTail赋值为新增的节点
                        }
                        // 9.2 如果e的hash值与老表的容量进行与运算为1,则扩容后的索引位置为:老表的索引位置＋oldCap
                        else {
                            if (hiTail == null) // 如果hiTail为空, 代表该节点为第一个节点
                                hiHead = e; // 则将hiHead赋值为第一个节点
                            else
                                hiTail.next = e;    // 否则将节点添加在hiTail后面
                            hiTail = e; // 并将hiTail赋值为新增的节点
                        }
                    } while ((e = next) != null);
                    // 10.如果loTail不为空（说明老表的数据有分布到新表上“原索引位置”的节点），则将最后一个节点
                    // 的next设为空，并将新表上索引位置为“原索引位置”的节点设置为对应的头节点
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 11.如果hiTail不为空（说明老表的数据有分布到新表上“原索引+oldCap位置”的节点），则将最后
                    // 一个节点的next设为空，并将新表上索引位置为“原索引+oldCap”的节点设置为对应的头节点
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 12.返回新表
    return newTab;
}
```

### rehash 计算旧表key在新表的位置


在拓容的时候，一般把长度拓为原来的2倍，所以元素的位置要么在原位置，要么在原位置移动2次幂的位置。**即要么在原位置，要么在原位置+旧长度**


![NCb7Bq.png](https://s1.ax1x.com/2020/06/15/NCb7Bq.png)

如图n是table的长度，当n拓展为2倍后，key1，key2的hash仍不变，但是**参与计算新下标hash范围了一位**，就是根据这个新增的bit，来取决于新下标，若为0，则与旧下标一致，若为1，则为 旧下标 + 旧长度

下方代码是计算key在HashMap数组下标的方法
```
// 代码1
static final int hash(Object key) { // 计算key的hash值
    int h;
    // 1.先拿到key的hashCode值; 2.将hashCode的高16位参与运算
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
// 代码2
// 将计算得到的hashcode与tabel.length - 1进行 与操作
static int indexFor(int h, int length) {  
   return h & (length-1);  
}  
```


## HashMap死循环

在JDK1.7版本及以前，多线程情况下使用HashMap可能会造成死循环

```
# 1.7版本的HashMap拓容
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}

```

原因就出在transfer方法中

### 案例分析

【博客原文】 https://www.jianshu.com/p/a105a7917d6f
https://my.oschina.net/xiaohai945/blog/1021838

<font color="red">**在1.7版本中，新插入的节点是在链表头部**</font>

<font color="red">**在1.8版本中，新插入的节点是在链表尾部**</font>

**原有HashMap**

![NiyUMD.png](https://s1.ax1x.com/2020/06/16/NiyUMD.png)

##### 单线程下的拓容

![Ni6kOe.png](https://s1.ax1x.com/2020/06/16/Ni6kOe.png)

![Ni6Wp6.png](https://s1.ax1x.com/2020/06/16/Ni6Wp6.png)

![Ni6mFI.png](https://s1.ax1x.com/2020/06/16/Ni6mFI.png)


#### 线程并发操作拓容

假设线程1执行到Entry<K, V> next = e.next; 时被操作系统调度挂起，而线程2完成了拓容操作

![NicHrF.png](https://s1.ax1x.com/2020/06/16/NicHrF.png)

当时间片重新切换到线程1时，e仍然指向key3，next指向key7,接下来线程1继续执行，<font color="red">**此时a循环的是已经拓容好的新的HashMap**</font>

![NiWoFJ.png](https://s1.ax1x.com/2020/06/16/NiWoFJ.png)

![NiWLy6.png](https://s1.ax1x.com/2020/06/16/NiWLy6.png)

![NifSFH.png](https://s1.ax1x.com/2020/06/16/NifSFH.png)

![NifF6P.png](https://s1.ax1x.com/2020/06/16/NifF6P.png)

<font color="red">**JDK1.8版本中采用loHead和loTail以及hiHead和hiTail两对头尾指针来规避多线程插入时的死锁问题**</font>

## HashMap节点丢失

这个问题同样是由于多线程并发操作HashMap引起的

假设有下面HashMap需要拓容

![NFeGr9.png](https://s1.ax1x.com/2020/06/16/NFeGr9.png)

正常拓容后应该是这样的

![NFesrd.png](https://s1.ax1x.com/2020/06/16/NFesrd.png)

但是此时有两个线程同时操作，线程1先操作7，然后放5，此时CPU时间片耗尽，挂起线程1

此时切换到线程2继续操作，但是e指向节点5,5->next = null结束循环，因此拓容结束，元素丢失

把一个线程非安全的集合作为全局共享的，本身就是一种错误的做法，并发下一定会产生错误。所以，解决问题的方法有两种

<font color="red">**1. 使用Collections.synchronizedMap(Mao<K,V> m)方法把HashMap变成一个线程安全的Map**</font>

<font color="red">**2. 使用Hashtable、ConcurrentHashMap这两个线程安全的Map**</font>

# HashMap面试总结

    HashMap1.7头插法和1.8中尾插法的原因
    
头插法是操作速度最快的，不需要进行遍历，找到数组位置就可以直接插入数据了。但是这种插入方法在并发场景下多线程操作同时拓容会出现循环列表。故1.8中采用了尾插法，但是同时带来了链表遍历的消耗，所以在1.8之后，HashMap添加了红黑树，当链表长度大于8时，会转变为红黑树

    HashMap为什么要改成“数组 + 链表 + 红黑树”
    
主要是为了提升还hash冲突时链表过长的查找性能，因为1.8使用的是尾插法，使用链表的查找性能是O(n), 而红黑树是O(logN)

    什么时候用链表，什么时候用数组
    
在插入的场景下，默认是使用链表的。当**同一个索引位置的节点数在新增后达到9个（阀值8，TREEIFY_THRESHOLD常量）**，则会触发链表节点转化为红黑树节点treeifyBin方法；而如果这是数组长度小于64，则不会进行转化，而是会拓容。

在移除的场景下，**当同一个索引位置的节点数在移除后达到6个**，且该索引位置的节点为红黑树节点，会触发红黑树节点转链表节点（untreeify）

    为什么链表转红黑树的阀值是8

阀值8是时间和空间上权衡的结果

红黑树节点大小约为链表节点的2倍，在节点太少时，红黑树的查找性能优势并不明显，付出2倍空间的代价作者觉得不值得。

理想情况下，使用随机的哈希码，节点分布在 hash 桶中的频率遵循泊松分布，按照泊松分布的公式计算，链表中节点个数为8时的概率为 0.00000006，这个概率足够低了，并且到8个节点时，红黑树的性能优势也会开始展现出来，因此8是一个较合理的数字。

    为什么红黑树转链表的阀值是6

如果我们设置节点多于8个转红黑树，少于8个就马上转链表，当节点个数在8徘徊时，就会频繁进行红黑树和链表的转换，造成性能的损耗。

    HashMap的threshold有什么用
    
是拓容阀值，默认是（capacity * loadFactory = 12）。在新建HashMap对象时，threshold还会被用来存初始化时的容量。HashMap直到我们第一次插入节点时，才会对table进行初始化，避免不必要的空间浪费

    HashMap默认初始容量是多少，有什么限制
    
初始容量是16，最大是1 >> 30, 且HashMap的长度必须是2的N次幂
这是通过以下方法实现的
```
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

|= 或等于，类似于 += ，a |= b 可以转化为 a = a|b

![NFhiJH.png](https://s1.ax1x.com/2020/06/16/NFhiJH.png)

暂且忽视 cap - 1 ，假设n的值为 0010 0001，则上述公式计算如图

![NF5oeP.png](https://s1.ax1x.com/2020/06/16/NF5oeP.png)



这五个公式会通过最高位的1，拿到2个1、4个1、8个1、 16个1。当然这取决于入参的大小，但是经过计算，拿到的必然是**低位全为1的值**。最后返回的时候 + 1，则会得到一个比n大2的N次方。

这时回头看  cap - 1 ，这时为了处理n本身是2的N次幂的情况

    HashMap的容量为什么是2的N次幂

1. HashMap的索引计算公式是 (n - 1) & hash，**当n是2的N次幂时，n-1 的低位全是1**。此时任何值跟 （n-1）进行 与运算会等于其本身，达到取模运算的效果，实现了均匀分布。
2. 当容量为2的N次幂时，hash冲突会减到最小


    HashMap负载因子是什么，默认值多少，为什么

负载因子loadFactory，用于计算HashMap的拓容阀值threshold，**拓容阀值 = 容量 * 负载因子**，默认值是0.75；

当负载因子较高时，会减少空间开销，但是hash冲突的概率会增大，增加查找成本；而如果较低，如0.5，此时hash冲突会降低，但是又会浪费一半空间

    HashMap中key的hash值怎么计算的

如下代码实现，拿到key的hashcode，并和自身的高16位做异或运算
```
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
让高位也参与运算，是为了在table的长度较少时，减少hash冲突

例如下图，如果不加入高位运算，由于 n-1 = 0000 0111, hash的结果只取决于hashcode的低三位，这样hash冲突的机会极大

![NkpdAg.png](https://s1.ax1x.com/2020/06/16/NkpdAg.png)

若高位参与运算，则索引结果就不会仅取决于低位

![NkpcuV.png](https://s1.ax1x.com/2020/06/16/NkpcuV.png)

    JDK1.8对HashMap的优化
    
1. 底层数据结构由 “数组 + 链表” 变为 “数组 + 链表 + 红黑树”，主要是优化了Hash冲突严重时，链表过长的查找性能 O(n) -> O(logn)
2. 计算table初始容量的方式发生改变，老方法是从1不断左移位运算，直到找到大于或等于入参的值。新方法是通过“五个位移 + 或运算”来计算
3. 优化了hash值的计算

```
// JDK 1.7.0
static int hash(int h) {
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
// JDK 1.8.0_191
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

4. 节点插入时从“头插法”改为“尾插法”
5. 扩容时计算节点在新表的索引位置方式从“h & (length-1)”改成“hash & oldCap”，性能可能提升不大，但设计更巧妙、更优雅。

## 其他Map使用场景总结

![NkPqbV.png](https://s1.ax1x.com/2020/06/16/NkPqbV.png)