---
title: 集合HashMap详细分析
date: 2019-09-20 15:02:46
author: MZK
img: http://static.mzkai.com/Blog/Map/map_bg_02.jpg
categories: 技术栈
tags:
  - 集合
  - 数据结构
---
# Map简介
Java 自带了各种 Map 类，这些 Map 类可归为三种类型：
1.通用Map：
&nbsp;&nbsp;&nbsp;&nbsp; 用于在应用程序中管理映射，通常在 java.util 程序包中实现 HashMap、Hashtable、Properties、LinkedHashMap、IdentityHashMap、TreeMap、WeakHashMap、ConcurrentHashMap
2.专用Map：
&nbsp;&nbsp;&nbsp;&nbsp; 通常我们不必亲自创建此类Map，而是通过某些其他类对其进行访问 java.util.jar.Attributes、javax.print.attribute.standard.PrinterStateReasons、java.security.Provider、java.awt.RenderingHints、javax.swing.UIDefaults
3.自行实现Map：
一个用于帮助我们实现自己的Map类的抽象类 AbstractMap
平时使用比较多的Map有：HashMap、TreeMap、Hashtable、LinkedHashMap
![](http://static.mzkai.com/Blog/Map/map02.jpg)
# HashMap
## 概述
HashMap是最常用的Map，它根据键的HashCode 值存储数据，实现Map接口的双列集合，数据结构是“链表散列”，也就是数组+链表 ，key唯一的value可以重复，根据键可以直接获取它的值，具有很快的访问速度。HashMap最多只允许一条记录的键为Null(多条会覆盖)，允许多条记录的值为 Null，非同步的。
![](http://static.mzkai.com/Blog/Map/map01.png)
本篇文章所分析的源码版本为 JDK 1.8。与 JDK 1.7 相比，JDK 1.8 对 HashMap 进行了一些优化。比如引入红黑树解决过长链表效率低的问题。重写 resize 方法，移除了 alternative hashing 相关方法，避免重新计算键的 hash 等。
其底层的数据结构为数组 + 链表 + 红黑树，当链表长度达到 TREEIFY_THRESHOLD = 8 时，该链表会自动转化为红黑树，以提升 HashMap 的查询、插入效率，它实现了 Map<K, V>，Cloneable，Serializable接口。
## 基本源码
 **Node<K,V> 类：**
```java
 1    // Node<K,V> 类用来实现数组及链表的数据结构
 2 　 static class Node<K,V> implements Map.Entry<K,V> {
 3         final int hash; //保存节点的 hash　值
 4         final K key; //保存节点的　key　值
 5         V value;　//保存节点的　value 值
 6         Node<K,V> next;　//指向链表结构下的当前节点的　next 节点，红黑树　TreeNode　节点中也有用到
 7 
 8         Node(int hash, K key, V value, Node<K,V> next) {
 9             this.hash = hash;
10             this.key = key;
11             this.value = value;
12             this.next = next;
13         }
14 
15         public final K getKey()        { }
16         public final V getValue()      {  }
17         public final String toString() { }
18 
19         public final int hashCode() {           
20         }
21 
22         public final V setValue(V newValue) {          
23         }
24 
25         public final boolean equals(Object o) {            
26         }
27     }
28     
29     public class LinkedHashMap<K,V> {
30           static class Entry<K,V> extends HashMap.Node<K,V> {
31                 Entry<K,V> before, after;
32                 Entry(int hash, K key, V value, Node<K,V> next) {
33                     super(hash, key, value, next);
34                 }    
35             }
36     }    
37     
38 　// TreeNode<K,V> 继承 LinkedHashMap.Entry<K,V>，用来实现红黑树相关的存储结构
39     static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
40         TreeNode<K,V> parent;  // 存储当前节点的父节点
41         TreeNode<K,V> left;　//存储当前节点的左孩子
42         TreeNode<K,V> right;　//存储当前节点的右孩子
43         TreeNode<K,V> prev;    // 存储当前节点的前一个节点
44         boolean red;　// 存储当前节点的颜色（红、黑）
45         TreeNode(int hash, K key, V val, Node<K,V> next) {
46             super(hash, key, val, next);
47         }
48 }
```
**HashMap 各常量、成员变量**:
> //默认的初始容量16,且实际容量是2的整数幂
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    //最大容量(传入容量过大将被这个值替换)
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认加载因子为0.75(当表达到3/4满时,才会再散列),这个因子在时间和空间代价之间达到了平衡.更高的因子可以降低表所需的空间,但是会增加查找代价,而查找是最频繁操作
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    //桶的树化阈值：即 链表转成红黑树的阈值，在存储数据时，当链表长度 >= 8时，则将链表转换成红黑树
    static final int TREEIFY_THRESHOLD = 8;
   // 桶的链表还原阈值：即 红黑树转为链表的阈值，当在扩容（resize（））时（HashMap的数据存储位置会重新计算），在重新计算存储位置后，当原有的红黑树内数量 <= 6时，则将 红黑树转换成链表
    static final int UNTREEIFY_THRESHOLD = 6;
>   //最小树形化容量阈值：即 当哈希表中的容量 > 该值时，才允许树形化链表 （即 将链表 转换成红黑树）

**三种构造方法**：
```java
    //默认构造方法
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
    //仅指定初始容量，装载因子的值采用默认的　0.75
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
    // 指定初始容量及装载因子
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
```
如果使用默认构造函数，则 HashMap 的初始化容量为 1 << 4 也就是 16，默认加载因子是 DEFAULT_LOAD_FACTOR = 0.75f，当 HashMap 底层的数组元素个数 > 数组容量 * 加载因子时，HashMap 将进行扩容操作，当然也可以在初始化时给定一个loadFactor。如果在初始化时给定 initialCapacity，则初始化容量 C 需满足：C 是 2 的幂次方且 C >= initialCapacity 且 C <= (1 << 30)。其中的 tableSizeFor 方法保证函数返回值是大于等于给定参数 initialCapacity 最小的 2 的幂次方的数值，具体为：
```java
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
## put及其相关方法
**put(K key, V value):**
```java
 1   // 指定节点 key,value，向 hashMap 中插入节点
 2 　public V put(K key, V value) {
 3     　  //注意待插入节点　hash 值的计算，调用了　hash(key) 函数
 4 　　    //调用 putVal（）进行节点的插入
 5         return putVal(hash(key), key, value, false, true);
 6     }
 7 　static final int hash(Object key) {
 8         int h;
 9 　　/*key的hash值的计算是通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销*/
10         return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
11     }
12 
13 　public void putAll(Map<? extends K, ? extends V> m) {
14         putMapEntries(m, true);
15     }
16 　
17 　/*把Map<? extends K, ? extends V> m 中的元素插入到　hashMap 中,若 evict 为 false,代表是在创建 hashMap 时调用了这个函数，例如利用上述构造函数３创建 hashMap;若 evict　为true,代表是在创建　hashMap 后才调用这个函数，例如上述的　putAll 函数。*/
18 
19 　final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
20         int s = m.size();
21         if (s > 0) {
22             /*如果是在创建 hashMap 时调用的这个函数则 table 一定为空*/
23             if (table == null) { 
24 　　　　  //根据待插入的map 的 size 计算要创建的　hashMap 的容量。
25                 float ft = ((float)s / loadFactor) + 1.0F;
26                 int t = ((ft < (float)MAXIMUM_CAPACITY) ?
27                          (int)ft : MAXIMUM_CAPACITY);
28 　　　　  //把要创建的　hashMap 的容量存在　threshold　中
29                 if (t > threshold)
30                     threshold = tableSizeFor(t);
31             }
32 　　　 //判断待插入的　map 的 size,若　size 大于　threshold，则先进行　resize()
33             else if (s > threshold)
34                 resize();
35             for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
36                 K key = e.getKey();
37                 V value = e.getValue();
38                 //实际也是调用　putVal　函数进行元素的插入
39                 putVal(hash(key), key, value, false, evict);
40             }
41         }
42     }
43 　
44     final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
45                    boolean evict) {
46         Node<K,V>[] tab; Node<K,V> p; int n, i;
47         if ((tab = table) == null || (n = tab.length) == 0)
48             n = (tab = resize()).length;
49 　　 /*根据 hash 值确定节点在数组中的插入位置，若此位置没有元素则进行插入，注意确定插入位置所用的计算方法为　(n - 1) & hash,由于　n 一定是２的幂次，这个操作相当于
50 　hash % n */
51         if ((p = tab[i = (n - 1) & hash]) == null)
52             tab[i] = newNode(hash, key, value, null);
53         else {//说明待插入位置存在元素
54             Node<K,V> e; K k;
55 　　　 　　　　//比较原来元素与待插入元素的　hash 值和　key 值
56             if (p.hash == hash &&
57                 ((k = p.key) == key || (key != null && key.equals(k))))
58                 e = p;
59 　　　 　　　　//若原来元素是红黑树节点，调用红黑树的插入方法:putTreeVal
60             else if (p instanceof TreeNode)
61                 e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
62             else {//证明原来的元素是链表的头结点，从此节点开始向后寻找合适插入位置
63                 for (int binCount = 0; ; ++binCount) {
64                     if ((e = p.next) == null) {
65 　　　　　　　//找到插入位置后，新建节点插入
66                         p.next = newNode(hash, key, value, null);
67 　　　　　　　//若链表上节点超过TREEIFY_THRESHOLD - 1，将链表变为红黑树
68                         if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
69                             treeifyBin(tab, hash);
70                         break;
71                     }
72                     if (e.hash == hash &&
73                         ((k = e.key) == key || (key != null && key.equals(k))))
74                         break;
75                     p = e;
76                 }
77             }//end else
78             if (e != null) { // 待插入元素在　hashMap 中已存在
79                 V oldValue = e.value;
80                 if (!onlyIfAbsent || oldValue == null)
81                     e.value = value;
82                 afterNodeAccess(e);
83                 return oldValue;
84             }
85         }//end else
86         ++modCount;
87         if (++size > threshold)
88             resize();
89         afterNodeInsertion(evict);
90         return null;
91     }
```
**红黑树的插入方法putTreeVal:**
```java
 1        /*读懂这个函数要注意理解 hash 冲突发生的几种情况
 2          １、两节点　key 值相同（hash值一定相同），导致冲突
 3          ２、两节点　key 值不同，由于 hash 函数的局限性导致hash 值相同，冲突
 4 　　 　　 ３、两节点　key 值不同，hash 值不同，但 hash 值对数组长度取模后相同，冲突
 5        */
 6         final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
 7                                        int h, K k, V v) {
 8             Class<?> kc = null;
 9             boolean searched = false;
10             TreeNode<K,V> root = (parent != null) ? root() : this;
11 　　　　　　　 //从根节点开始查找合适的插入位置（与二叉搜索树查找过程相同）
12             for (TreeNode<K,V> p = root;;) {
13                 int dir, ph; K pk;
14                 if ((ph = p.hash) > h)
15                     dir = -1;　//　dir小于０，接下来查找当前节点左孩子
16                 else if (ph < h)
17                     dir = 1;　//　dir大于０，接下来查找当前节点右孩子
18                 else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
19 　　　　　　　　　　　　//进入这个else if 代表　hash 值相同，key　相同
20                     return p;
21 　　　　 　　　　 /*要进入下面这个else if,代表有以下几个含义:
22                   1、当前节点与待插入节点　key　不同,　hash 值相同
23     　　　　　　　　2、ｋ是不可比较的，即ｋ并未实现　comparable<K>　接口
　　　　　　　　　　　　　　（若 k 实现了comparable<K>　接口，comparableClassFor（k）返回的是ｋ的　class,而不是　null）
24                 　　或者　compareComparables(kc, k, pk)　返回值为 0
　　　　　　　　　　　　　　(pk 为空　或者　按照 k.compareTo(pk) 返回值为０，
　　　　　　　　　　　　　　返回值为０可能是由于　ｋ的compareTo 方法实现不当引起的，compareTo 判定相等，而上个 else if　中　equals 判定不等)*/
25                 else if ((kc == null &&
26                           (kc = comparableClassFor(k)) == null) ||
27                          (dir = compareComparables(kc, k, pk)) == 0) {
28                     //在以当前节点为根的整个树上搜索是否存在待插入节点（只会搜索一次）
29                     if (!searched) {
30                         TreeNode<K,V> q, ch;
31                         searched = true;
32                         if (((ch = p.left) != null &&
33                              (q = ch.find(h, k, kc)) != null) ||
34                             ((ch = p.right) != null &&
35                              (q = ch.find(h, k, kc)) != null))
36 　　　　　　　　　　　　　　　　　//若树中存在待插入节点，直接返回
37                             return q;
38                     }
39 　　　　　 　　　　　　 // 既然ｋ是不可比较的，那我自己指定一个比较方式
40                     dir = tieBreakOrder(k, pk);
41                 }//end else if
42 
43                 TreeNode<K,V> xp = p;
44                 if ((p = (dir <= 0) ? p.left : p.right) == null) {
45 　　　　　　　　　　　　//找到了待插入的位置，xp 为待插入节点的父节点
46 　　　　　　　　　　　　//注意TreeNode节点中既存在树状关系，也存在链式关系，并且是双端链表
47                     Node<K,V> xpn = xp.next;
48                     TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
49                     if (dir <= 0)
50                         xp.left = x;
51                     else
52                         xp.right = x;
53                     xp.next = x;
54                     x.parent = x.prev = xp;
55                     if (xpn != null)
56                         ((TreeNode<K,V>)xpn).prev = x;
57 　　　　　　　　　　　　//插入节点后进行二叉树的平衡操作
58                     moveRootToFront(tab, balanceInsertion(root, x));
59                     return null;
60                 }
61             }//end for
62         }//end putTreeVal    　
63 　　
64 　　 　　static int tieBreakOrder(Object a, Object b) {
65             int d;
66             //System.identityHashCode()实际是利用对象 a,b 的内存地址进行比较
67             if (a == null || b == null ||
68                 (d = a.getClass().getName().
69                  compareTo(b.getClass().getName())) == 0)
70                 d = (System.identityHashCode(a) <= System.identityHashCode(b) ?
71                      -1 : 1);
72             return d;
73         }
```
## get及其相关方法
**get(Object key):**
```java
 1     public V get(Object key) {
 2         Node<K,V> e;
 3 　　//实际上是根据输入节点的 hash 值和 key 值利用getNode 方法进行查找
 4         return (e = getNode(hash(key), key)) == null ? null : e.value;
 5     }
 6 　
 7 　final Node<K,V> getNode(int hash, Object key) {
 8         Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
 9         if ((tab = table) != null && (n = tab.length) > 0 &&
10             (first = tab[(n - 1) & hash]) != null) {
11             if (first.hash == hash && // always check first node
12                 ((k = first.key) == key || (key != null && key.equals(k))))
13                 return first;
14             if ((e = first.next) != null) {
15                 if (first instanceof TreeNode)
16 　　　　　　　　　　　　//若定位到的节点是　TreeNode 节点，则在树中进行查找
17                     return ((TreeNode<K,V>)first).getTreeNode(hash, key);
18                 do {//否则在链表中进行查找
19                     if (e.hash == hash &&
20                         ((k = e.key) == key || (key != null && key.equals(k))))
21                         return e;
22                 } while ((e = e.next) != null);
23             }
24         }
25         return null;
26     }
```
**红黑树的获取方法getTreeNode:**
```java
1         final TreeNode<K,V> getTreeNode(int h, Object k) {
 2 　　　　　　　　//从根节点开始，调用 find 方法进行查找
 3             return ((parent != null) ? root() : this).find(h, k, null);
 4         }
 5 　
 6         final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
 7             TreeNode<K,V> p = this;
 8             do {
 9                 int ph, dir; K pk;
10                 TreeNode<K,V> pl = p.left, pr = p.right, q;
11 　　　　　　　　　//首先进行hash 值的比较，若不同令当前节点变为它的左孩子或者右孩子
12                 if ((ph = p.hash) > h)
13                     p = pl;
14                 else if (ph < h)
15                     p = pr;
16 　　　　　　　　　//hash 值相同，进行 key　值的比较 
17                 else if ((pk = p.key) == k || (k != null && k.equals(pk)))
18                     return p;
19                 else if (pl == null)
20                     p = pr;
21                 else if (pr == null)
22                     p = pl;
23 　　　　　　　　　//执行到这儿，意味着hash 值相同，key 值不同 
24    　　　　　　　　//若k 是可比较的并且k.compareTo(pk) 返回结果不为０可进入下面elseif   
25                 else if ((kc != null ||
26                           (kc = comparableClassFor(k)) != null) &&
27                          (dir = compareComparables(kc, k, pk)) != 0)
28                     p = (dir < 0) ? pl : pr;
29                 /*若 k 是不可比较的　或者　k.compareTo(pk) 返回结果为０则在整棵树中进行查找，先找右子树，右子树没有再找左子树*/
30                 else if ((q = pr.find(h, k, kc)) != null)
31                     return q;
32                 else
33                     p = pl;
34             } while (p != null);
35             return null;
36         }
```
## resize扩容方法
**resize():**
```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
　　    /*
        1、resize（）函数在size　> threshold时被调用。
            oldCap大于 0 代表原来的 table 表非空， oldCap 为原表的大小，
            oldThr（threshold） 为 oldCap × load_factor
    　  */
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
　　    /*
        2、resize（）函数在table为空被调用。
			oldCap 小于等于 0 且 oldThr 大于0，代表用户创建了一个 HashMap，但是使用的构造函数为
			HashMap(int initialCapacity, float loadFactor) 或 HashMap(int initialCapacity)
			或 HashMap(Map<? extends K, ? extends V> m)，导致 oldTab 为 null，oldCap 为0，
			oldThr 为用户指定的 HashMap的初始容量。
    　  */
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
　　　  /*
            3、resize（）函数在table为空被调用。
            oldCap 小于等于 0 且 oldThr 等于0，用户调用 HashMap()构造函数创建的　HashMap，所有值均采用默认值，
       　　 oldTab（Table）表为空，oldCap为0，oldThr等于0，
    　　*/
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;        
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
　　　　　　　//把　oldTab 中的节点　reHash 到　newTab 中去
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
　　　　　　　　　　　　//若节点是单个节点，直接在 newTab　中进行重定位
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
　　　　　　　　　　　　//若节点是　TreeNode 节点，要进行 红黑树的 rehash　操作
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
　　　　　　　　　　　　//若是链表，进行链表的 rehash　操作
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
            　　                next = e.next;
　　　　　　　　　　　　　　　　　　//根据算法　e.hash & oldCap　判断节点位置　rehash　后是否发生改变
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
　　　　　　　　　　　　　　　　// rehash　后节点新的位置一定为原来基础上加上　oldCap
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
**红黑树rehash:**
```java
 1     //这个函数的功能是对红黑树进行　rehash 操作
 2     final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
 3             TreeNode<K,V> b = this;
 4             // Relink into lo and hi lists, preserving order
 5             TreeNode<K,V> loHead = null, loTail = null;
 6             TreeNode<K,V> hiHead = null, hiTail = null;
 7             int lc = 0, hc = 0;
 8     　　　　　//由于　TreeNode 节点之间存在双端链表的关系，可以利用链表关系进行 rehash
 9             for (TreeNode<K,V> e = b, next; e != null; e = next) {
10                 next = (TreeNode<K,V>)e.next;
11                 e.next = null;
12                 if ((e.hash & bit) == 0) {
13                     if ((e.prev = loTail) == null)
14                         loHead = e;
15                     else
16                         loTail.next = e;
17                     loTail = e;
18                     ++lc;
19                 }
20                 else {
21                     if ((e.prev = hiTail) == null)
22                         hiHead = e;
23                     else
24                         hiTail.next = e;
25                     hiTail = e;
26                     ++hc;
27                 }
28             }
29             
30             //rehash 操作之后注意对根据链表长度进行　untreeify　或　treeify　操作
31             if (loHead != null) {
32                 if (lc <= UNTREEIFY_THRESHOLD)
33                     tab[index] = loHead.untreeify(map);
34                 else {
35                     tab[index] = loHead;
36                     if (hiHead != null) // (else is already treeified)
37                         loHead.treeify(tab);
38                 }
39             }
40             if (hiHead != null) {
41                 if (hc <= UNTREEIFY_THRESHOLD)
42                     tab[index + bit] = hiHead.untreeify(map);
43                 else {
44                     tab[index + bit] = hiHead;
45                     if (loHead != null)
46                         hiHead.treeify(tab);
47                 }
48             }//end if
49         }//end split
50```

关于　HashMap 源码阅读的相关知识就先做到这样了，有一些地方还需要更深入理解透彻（例如红黑树的插入节点之后的平衡操作，删除节点操作），大家一起共同学习进步。

   
 

