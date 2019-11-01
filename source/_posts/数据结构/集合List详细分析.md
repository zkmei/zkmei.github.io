---
title: 集合List详细分析
date: 2019-07-22 23:33:10
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/List/post-bg-list.jpg
categories: 技术栈
tags:
  - 数据结构
  - 集合
  - list
---
# 集合——List
## 简介
之前介绍了集合Set，在这章中我们主要介绍Collection的其中一种实现方式，List。 
> List主要分为3类，ArrayList， LinkedList和Vector

## 继承Collection

![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/List/post_list_01.png)

List是一个有序的集合，和set不同的是，List允许存储项的值为空，也允许存储相等值的存储项，还举了e1.equal(e2)的例子。
List是继承于Collection接口，除了Collection通用的方法以外，扩展了部分只属于List的方法。

![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/List/post_list_02.png)
&nbsp;&nbsp;&nbsp;&nbsp;List比Collection主要多了几个add（…）方法和remove(…)方法的重载，还有几个index(…)， get(…)方法。而AbstractList也只是实现了List接口部分的方法，和AbstractCollection是一个思路。

## ArrayList
ArrayList是一个数组实现的列表，由于数据是存入数组中的，所以它的特点也和数组一样，查询很快，但是中间部分的插入和删除很慢。
### ArrayList的类关系和成员变量
```java
/ArrayList继承了Serializable并且申明了serialVersionUID，表示ArrayList是一个可序列化的对象，可以用Bundle传递
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    private static final long serialVersionUID = 8683452581122892189L;
    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 10;
    
    // 用于空实例的共享空数组实例
    private static final Object[] EMPTY_ELEMENTDATA = {};
    
    //用于默认大小的空实例的共享空数组实例。我们将其与空的ELEMENTDATA区别开来，以了解在添加第一个元素时应该膨胀多少。
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    
    //从这里可以看到，ArrayList的底层是由数组实现的，并且默认数组的默认大小是10
    transient Object[] elementData;

    //ArrayList的大小(它包含的元素的数量)
    private int size;
```
### ArrayList的构造函数
```java
    /**
     * 构造一个具有指定初始容量的空列表。
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 构造一个初始容量为10的空列表。
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 构造一个包含指定集合的元素的列表，按集合的迭代器返回元素的顺序排列
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```
### add
```java
    //指定index执行add操作，还是在尾部执行add操作，都会先确认当前的数组空间是否够插入数据
    //int oldCapacity = elementData.length;
    //int newCapacity = oldCapacity + (oldCapacity >> 1);
    //if (newCapacity - minCapacity < 0)newCapacity = minCapacity;
    //ArrayList默认每次都是自增50%的大小再和minCapacity比较，如果还是不够，就把size扩充至minCapacity
    //然后，如果是队尾插入，也简单，就把数组向后移动一位，然后赋值
    //如果是在中间插入，需要用到System.arraycopy，把index开始所有数据向后移动一位
    //再进行插入
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    /**
     * 将指定元素插入到列表中的指定位置。将当前位于该位置的元素(如果有)和任何后续元素向右移动(将一个元素添加到它们的索引中)。
     *
     * @param index index at which the specified element is to be inserted
     * @param element element to be inserted
     * @throws IndexOutOfBoundsException {@inheritDoc}
     */
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
    private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }
    
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    /**
     * 增加容量，以确保它至少可以容纳由最小容量参数指定的元素数量。
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
### remove
```java
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }

    /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>
     * (if such an element exists).  Returns <tt>true</tt> if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return <tt>true</tt> if this list contained the specified element
     */
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```
### indeOf
```java
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```
## Vector
Vector 可实现自动增长的对象数组。 
java.util.vector提供了向量类(Vector)以实现类似动态数组的功能。 
创建了一个向量类的对象后，可以往其中随意插入不同类的对象，即不需顾及类型也不需预先选定向量的容量，并可以方便地进行查找。

对于预先不知或者不愿预先定义数组大小，并且需要频繁地进行查找，插入，删除工作的情况，可以考虑使用向量类。

向量类提供了三种构造方法：
> public vector() 
> public vector(int initialcapacity,int capacityIncrement) 
> public vector(int initialcapacity)

Vector大致与ArrayList一致，但是有以下几点区别

1.初始化
默认无参构造方法 Vector会初始化一个长度为10的数组，ArrayList在具体调用时再创建数组。
比较之下，ArrayList延迟化加载更节省空间
2.扩容 (grow())
Vector当增量为0时，扩充为原来大小的2倍，当增量大于0时，扩充为原来大小加增量 ArrayList扩充算法：原来数组大小加原来数组的一半
3.线程安全
Vector是线程安全的，ArrayList是非线程安全的
Vector的线程安全包括其内部类如迭代器实现类ListItr

其实最大的区别就是线程安全性，当然如果我们想创建一个线程安全的ArrayList，可以调用Collections.synchronizedList()，得到静态内部类SynchronizedList，利用同步代码块处理ArrayList。
这种方式得到的线程安全的ArrayList和Vector有什么区别？很简单，一是同步代码块和同步方法的区别，剩下的是ArrayList和Vector除了线程安全性的其他区别；还有一点不能忽略，前者的迭代器的同步实现需要使用者手动控制

## LinkedList
LinkedList也是List接口的实现类，与ArrayList不同之处是采用的存储结构不同，ArrayList的数据结构为线性表，而LinkedList数据结构是链表。链表数据结构的特点是每个元素分配的空间不必连续、插入和删除元素时速度非常快、但访问元素的速度较慢。

LinkedList是一个双向链表, 当数据量很大或者操作很频繁的情况下，添加和删除元素时具有比ArrayList更好的性能。但在元素的查询和修改方面要弱于ArrayList。LinkedList类每个结点用内部类Node表示，LinkedList通过first和last引用分别指向链表的第一个和最后一个元素，当链表为空时，first和last都为NULL值。
> LinkedList 是一个继承于AbstractSequentialList的双向链表。
> LinkedList 可以被当作堆栈、队列或双端队列进行操作。
> LinkedList 实现 List 接口，所以能对它进行队列操作。
> LinkedList 实现 Deque 接口，能将LinkedList当作双端队列使用。
> LinkedList 实现了Cloneable接口，即覆盖了函数clone()，能克隆。
> LinkedList 实现java.io.Serializable接口，所以LinkedList支持序列化，能通过序列化去传输。
> LinkedList 是非同步的。

Node节点一共有三个属性：item代表节点值，prev代表节点的前一个节点，next代表节点的后一个节点。每个结点都有一个前驱和后继结点，并且在 LinkedList中也定义了两个变量分别指向链表中的第一个和最后一个结点。
```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
    transient int size = 0;

    /**
     * 指向链表中的第一个节点
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * 指向链表中的最后一个节点
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
    
    ...
    
     private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

### add

LinkedList提供了多个添加元素的方法：
● 在链表尾部添加一个元素，如果成功，返回true，否则返回false boolean add(E e)
● 在链表头部插入一个元素 void addFirst(E e)
● 在链表尾部添加一个元素 addLast(E e)
● 在指定位置插入一个元素 void add(int index, E element)
```java
     private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
    
    public void add(int index, E element) {
        checkPositionIndex(index);

        if (index == size)
            linkLast(element);
        else
            linkBefore(element, node(index));
    }
    
    /**
     * Links e as last element.
     */
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }

    /**
     * Inserts element e before non-null Node succ.
     */
    void linkBefore(E e, Node<E> succ) {
        // assert succ != null;
        final Node<E> pred = succ.prev;
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;
        if (pred == null)
            first = newNode;
        else
            pred.next = newNode;
        size++;
        modCount++;
    }

```
### remove
LinkedList提供了多个删除元素的方法：
● 从当前链表中移除指定的元素 boolean remove(Object o) 
● 从当前链表中移除指定位置的元素 E remove(int index)
● 从当前链表中移除第一个元素 E removeFirst()
● 从当前链表中移除最后一个元素 E removeLast()
● 从当前链表中移除第一个元素，同removeLast()相同 E remove()






## 小结
通过上面对ArrayList和LinkedList的分析，可以理解List的3个特性 
1.是按顺序查找 
2.允许存储项为空 
3.允许多个存储项的值相等 
可以知其然知其所以然

然后对比LinkedList和ArrayList的实现方式不同，可以在不同的场景下使用不同的List 
ArrayList是由数组实现的，方便查找，返回数组下标对应的值即可，适用于多查找的场景 
LinkedList由链表实现，插入和删除方便，适用于多次数据替换的场景