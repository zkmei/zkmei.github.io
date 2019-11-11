---
title: ConcurrentHashMap——JDK1.8版本
date: 2019-09-23 24:02:46
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Map/post-bg-2019.jpg
categories: 技术栈
top: true
tags:
  - 集合
  - 数据结构
---
# ConcurrentHashMap简介
ConcurrentHashMap从JDK1.5开始随java.util.concurrent包一起引入JDK中，主要为了解决HashMap线程不安全和Hashtable效率不高的问题。众所周知，HashMap在多线程编程中是线程不安全的，而Hashtable由于使用了synchronized修饰方法而导致执行效率不高；因此，在concurrent包中，实现了ConcurrentHashMap以使在多线程编程中可以使用一个高性能的线程安全HashMap方案。

而JDK1.7之前的ConcurrentHashMap使用分段锁机制实现，JDK1.8则使用数组+链表+红黑树数据结构和CAS原子操作实现ConcurrentHashMap；本文将介绍JDK8版本的实现方案，[JDK1.7版本链接](http://www.aikeke503.cn/2019/07/06/ji-zhu-zhan/shu-ju-jie-gou/shu-ju-jie-gou-zhi-dui-lie-queue/)
## CAS原理
一般地，锁分为悲观锁和乐观锁：悲观锁认为对于同一个数据的并发操作，一定是为发生修改的；而乐观锁则任务对于同一个数据的并发操作是不会发生修改的，在更新数据时会采用尝试更新不断重试的方式更新数据。

CAS（Compare And Swap，比较交换）：CAS有三个操作数，内存值V、预期值A、要修改的新值B，当且仅当A和V相等时才会将V修改为B，否则什么都不做。Java中CAS操作通过JNI本地方法实现，在JVM中程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（Lock Cmpxchg）；反之，如果程序是在单处理器上运行，就省略lock前缀。

Intel的手册对lock前缀的说明如下：
确保对内存的读-改-写操作原子执行。之前采用锁定总线的方式，但开销很大；后来改用缓存锁定来保证指令执行的原子性。
禁止该指令与之前和之后的读和写指令重排序。
把写缓冲区中的所有数据刷新到内存中。
CAS同时具有volatile读和volatile写的内存语义。

不过CAS操作也存在一些缺点：1. 存在ABA问题，其解决思路是使用版本号；2. 循环时间长，开销大；3. 只能保证一个共享变量的原子操作。
## 源码分析
JDK1.8的ConcurrentHashMap数据结构比JDK1.7之前的要简单的多，其使用的是HashMap一样的数据结构：数组+链表+红黑树。ConcurrentHashMap中包含一个table数组，其类型是一个Node数组；而Node是一个继承自Map.Entry<K, V>的链表，而当这个链表结构中的数据大于8，则将数据结构升级为TreeBin类型的红黑树结构。另外，JDK1.8中的ConcurrentHashMap中还包含一个重要属性sizeCtl，其是一个控制标识符，不同的值代表不同的意思：其为0时，表示hash表还未初始化，而为正数时这个数值表示初始化或下一次扩容的大小，相当于一个阈值；即如果hash表的实际大小>=sizeCtl，则进行扩容，默认情况下其是当前ConcurrentHashMap容量的0.75倍；而如果sizeCtl为-1，表示正在进行初始化操作；而为-N时，则表示有N-1个线程正在进行扩容。
### ConcurrentHashMap的初始化
```java
/**
 * 创建一个新的空映射，初始表大小基于根据给定的元素数量({@code initialCapacity})和初始表密度({@code loadFactor})
 * 
 * @param initialCapacity 初始化容量. 
 * @param loadFactor 加载因子
 * @param concurrencyLevel 预估并发度
 * @throws IllegalArgumentException 单元的初始容量为负或负载因子为非正
 * 
 */
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```
构造ConcurrentHashMap时并不会对hash表（Node<K, V>[] table）进行初始化，hash表的初始化是在插入第一个元素时进行的。在put操作时，如果检测到table为空或其长度为0时，则会调用initTable()方法对table进行初始化操作。
```java
/**
 * Initializes table, using the size recorded in sizeCtl.
 * 
 * sizeCtl<0则正在进行初始化
 * sizeCtl大于等于0，则使用CAS操作比较sizeCtl的值是否是-1，如果是-1则进行初始化；
 * 初始化时，如果sizeCtl的值为0，则创建默认容量的table，否则创建大小为sizeCtl的table，
 * 然后重置sizeCtl的值为0.75n，即当前table容量的0.75倍
 */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        //CAS操作比较sizeCtl的值是否是-1
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    //确认创建容量
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
###  链表和红黑树结构转换
```java
/**
 * 链表转换为红黑树
 */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        //首先判断hash表的大小是否大于等于MIN_TREEIFY_CAPACITY，默认值为64
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            //直接hash表扩容
            tryPresize(n << 1);
        //CAS获取指定的Node节点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}

该方法首先会检查hash表的大小是否大于等于MIN_TREEIFY_CAPACITY，默认值为64，如果小于该值，则表示不需要转化为红黑树结构，直接将hash表扩容即可。
如果当前table的长度大于64，则使用CAS获取指定的Node节点，然后对该节点通过synchronized加锁，由于只对一个Node节点加锁，因此该操作并不影响其他Node节点的操作，因此极大的提高了ConcurrentHashMap的并发效率。加锁之后，便是将这个Node节点所在的链表转换为TreeBin结构的红黑树。

```java
/**
 * 红黑树转换为链表
 */
static <K,V> Node<K,V> untreeify(Node<K,V> b) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = b; q != null; q = q.next) {
        Node<K,V> p = new Node<K,V>(q.hash, q.key, q.val, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```
### ConcurrentHashMap的操作
#### 1.get(Object key)
```java
public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        // hash再hash避免hash冲突
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            //获取Node
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            //遍历Node，返回value
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
}
```
首先spread()对key进行hash再hash，tabAt()找到对应的Node(链表或者红黑树结构)遍历数据,其中根据tabAn()中是sun.misc.Unsafe类中的getObjectVolatile()来获取Node。

#### 2.put(K key, V value)
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key和value均不能为null
    if (key == null || value == null) throw new NullPointerException();
    //hash再hash避免hash冲突
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            //初始化，然后再次进入循环获取Node节点的位置
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            //节点为null,casTabAT方法插入Node，此时不加锁
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        //MOVED=-1,即需要扩容
        else if ((fh = f.hash) == MOVED)
            //helpTransfer扩容方法
            tab = helpTransfer(tab, f);
        //其他情况插入Node,加锁
        else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    //最后记录元素数量
    addCount(1L, binCount);
    return null;
}
```
#### 3.size
JDK1.8的ConcurrentHashMap中保存元素的个数的记录方法也有不同，首先在添加和删除元素时，会通过CAS操作更新ConcurrentHashMap的baseCount属性值来统计元素个数。但是CAS操作可能会失败，因此，ConcurrentHashMap又定义了一个CounterCell数组来记录CAS操作失败时的元素个数。因此，ConcurrentHashMap中元素的个数则通过如下方式获得：
元素总数 = baseCount + sum(CounterCell)
JDK1.8中提供了两种方法获取ConcurrentHashMap中的元素个数:
```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
如代码所示，size只能获取int范围内的ConcurrentHashMap元素个数；而如果hash表中的数据过多，超过了int类型的最大值，则推荐使用mappingCount()方法获取其元素个数。

以上主要分析了ConcurrentHashMap在JDK1.8中的实现方案，当然ConcurrentHashMap的功能强大，还有很多方法本文都未能详细解析，但其分析方法与本文以上的内容类似，感兴趣的同学可以自行分析比较。