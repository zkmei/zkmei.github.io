---
title: 深入理解JVM（二）——垃圾回收
date: 2019-09-10 19:30:33
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/JVM/GC/jvm-gc.jpg
categories: 后端
tags:
    - JVM
    - GC
---
# 概述
GC（垃圾收集器）需要完成的三件事情：
- 哪些内存需要回收？
- 什么时候回收？
- 如何回收？

# 对象“已死”
在堆里面存放着Java所有的对象实例，垃圾收集器在对堆进行回收前要先确定判断哪些还“存活”着，哪些已经“死去”（你能再被任何途径使用的对象）。判断对象是否存活的算法主要有：
**引用计数算法**：对象的引用计数器，有一个地方引用就+1，引用失效就-1。
**可达性分析算法**: "GC Roots"的对象作为起点，向下搜索，当一个对象到GC Roots没有任何引用链相连，则为可回收对象。
在Java中可作为GC Roots的对象包括：
- 虚拟机栈（栈帧中的本地变量表）中的引用对象。
- 方法区中类静态熟悉引用的对象。
- 方法区中常量引用的对象。
- 本地房发展中JNI（Native方法）引用的对象。

## 引用
从上面可以看出判断对象是否存活都与“引用”有关。引用的定义：JDK 1.2 之前reference类型的数据中存储的数值代表的是另一块内存的起始地址，这块内存代表的就是一个引用。JDK 1.2之后将引用分为强引用，软引用，弱引用，虚引用，四种引用强度依次逐渐减弱。
- **强引用（StrongReference）**：只要强引用存在，垃圾回收器将永远不会回收被引用的对象，哪怕内存不足时，JVM也会直接抛出OutOfMemoryError，不会去回收。如果想中断强引用与对象之间的联系，可以显示的将强引用赋值为null，这样一来，JVM就可以适时的回收对象了
```java
    Object obj = new Object(); //只要obj还指向Object对象，Object对象就不会被回收
    obj = null;  //手动置null
```
- **软引用（SoftReference）**：描述还有用但并非必须的对象。在内存将要溢出前，软引用对象列入回收中进行二次回收。
- **弱引用（WeakReference）**：描述还有用但并非必须的对象，强度比软引用弱，只能生存到下一次垃圾收集之前。
```java
/**
 * Class: People
 * Param: String name， int age，
 */
public class test{
    public static void main(String[] args) {  
        WeakReference<People>reference=new WeakReference<People>(new People("zhouqian",20));  
        System.out.println(reference.get());  
        System.gc();//通知GVM回收资源  
        System.out.println(reference.get());  
    }  
}  
//输出结果：
//[name:zhouqian,age:20]
//null
```
被弱引用关联的对象是指只有弱引用与之关联，如果存在强引用同时与之关联，则进行垃圾回收时也不会回收该对象（软引用也是如此）。
```java
/**
 * Class: People
 * Param: String name， int age，
 */
public class test{
    public static void main(String[] args) {  
        People people=new People("zhouqian",20);  
        WeakReference<People>reference=new WeakReference<People>(people);//关联强引用
        System.out.println(reference.get());  
        System.gc();//通知GVM回收资源  
        System.out.println(reference.get());  
    }  
}  
//输出结果：
//[name:zhouqian,age:20]
//null
```
- **虚引用（PhantomReference）**：虚引用是最弱的一种引用关系,它随时可能会被回收。
```java
public class Main {  
    public static void main(String[] args) {  
        ReferenceQueue<String> queue = new ReferenceQueue<String>();  
        PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);  
        System.out.println(pr.get());  
    }  
}
```
## 生存还是死亡
即使在可达性分析算法中不可达对象也不一定是“非死不可”，标记为“死亡”必须经历两个过程：如果对象在进行可达性分析后发现没有雨GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选条件为是否有必要执行finalize（）方法。当对象没有覆盖finalize()方法，或者finalize()方法已被虚拟机调用过，虚拟机则视这两种情况为“没有必要执行”。
```java
public class FinalizeEscapeGC {

    public static FinalizeEscapeGC SAVE_HOOK = null;

    public void isAlive(){
        System.out.println("i`m still alive ！");
    }

    @Override
    protected void finalize() throws Throwable{
        super.finalize();
        System.out.println("finalize method executed!");
        FinalizeEscapeGC.SAVE_HOOK = this;
    }

    public static void main(String[] args) throws Throwable {
        SAVE_HOOK = new FinalizeEscapeGC();

        //对象第一次成功拯救自己
        SAVE_HOOK = null;
        System.gc();
        //因为finalize()方法优先级低，暂停0.5s 等待
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("i`m dead ！");
        }

        //下面代码与上面代码完全相同，但自救失败
        SAVE_HOOK = null;
        System.gc();
        //因为finalize()方法优先级低，暂停0.5s 等待
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("i`m dead :(");
        }
    }
}
    \\运行结果：
    \\finalize method executed!
    \\i`m still alive ！
    \\i`m dead ！
```
## 垃圾收集算法
### 1.标记-清除算法
标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。
在标记阶段首先通过根节点(GC Roots)，标记所有从根节点开始的对象，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。
![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/JVM/GC/jvm-gc01.jpg)
适用场合：
- 存活对象较多的情况下比较高效
- 适用于年老代（即旧生代）
缺点：
- 容易产生内存碎片，再来一个比较大的对象时（典型情况：该对象的大小大于空闲表中的每一块儿大小但是小于其中两块儿的和），会提前触发垃圾回收
- 扫描了整个空间两次（第一次：标记存活对象；第二次：清除没有标记的对象）
### 2.复制算法
从根集合节点进行扫描，标记出所有的存活对象，并将这些存活的对象复制到一块儿新的内存（图中下边的那一块儿内存）上去，之后将原来的那一块儿内存（图中上边的那一块儿内存）全部回收掉
![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/JVM/GC/jvm-gc02.jpg)
现在的商业虚拟机都采用这种收集算法来回收新生代。
适用场合：
- 存活对象较少的情况下比较高效
- 扫描了整个空间一次（标记存活对象并复制移动）
- 适用于年轻代（即新生代）：基本上98%的对象是”朝生夕死”的，存活下来的会很少
缺点：
- 需要一块儿空的内存空间
- 需要复制移动对象
### 3.标记-整理算法
标记-压缩算法是一种老年代的回收算法，它在标记-清除算法的基础上做了一些优化。
首先也需要从根节点开始对所有可达对象做一次标记，但之后，它并不简单地清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。这种方法既避免了碎片的产生，又不需要两块相同的内存空间，因此，其性价比比较高。
![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/JVM/GC/jvm-gc03.jpg)
### 4.分代收集算法
分代收集算法是目前大部分JVM的垃圾收集器采用的算法。它的核心思想是根据对象存活的生命周期将内存划分为若干个不同的区域。一般情况下将堆区划分为老年代（Tenured Generation）和新生代（Young Generation），在堆区之外还有一个代就是永久代（Permanet Generation）。老年代的特点是每次垃圾收集时只有少量对象需要被回收，而新生代的特点是每次垃圾回收时都有大量的对象需要被回收，那么就可以根据不同代的特点采取最适合的收集算法。
#### 4.1 年轻代（Young Generation）的回收算法 (回收主要以Copying为主)
a) 所有新生成的对象首先都是放在年轻代的。年轻代的目标就是尽可能快速的收集掉那些生命周期短的对象。
b) 新生代内存按照8:1:1的比例分为一个eden区和两个survivor(survivor0,survivor1)区。一个Eden区，两个 Survivor区(一般而言)。大部分对象在Eden区中生成。回收时先将eden区存活对象复制到一个survivor0区，然后清空eden区，当这个survivor0区也存放满了时，则将eden区和survivor0区存活对象复制到另一个survivor1区，然后清空eden和这个survivor0区，此时survivor0区是空的，然后将survivor0区和survivor1区交换，即保持survivor1区为空(美团面试，问的太细，为啥保持survivor1为空，答案：为了让eden和survivor0 交换存活对象)， 如此往复。当Eden没有足够空间的时候就会 触发jvm发起一次Minor GC
c) 当survivor1区不足以存放 eden和survivor0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次Full GC(Major GC)，也就是新生代、老年代都进行回收。
d) 新生代发生的GC也叫做Minor GC，MinorGC发生频率比较高(不一定等Eden区满了才触发)。
#### 4.2 年老代（Old Generation）的回收算法（回收主要以Mark-Compact为主）
a) 在年轻代中经历了N次垃圾回收后仍然存活的对象，就会被放到年老代中。因此，可以认为年老代中存放的都是一些生命周期较长的对象。
b) 内存比新生代也大很多(大概比例是1:2)，当老年代内存满时触发Major GC即Full GC，Full GC发生频率比较低，老年代对象存活时间比较长，存活率标记高。
#### 4.3 持久代（Permanent Generation）(也就是方法区)的回收算法
  用于存放静态文件，如Java类、方法等。持久代对垃圾回收没有显著影响，但是有些应用可能动态生成或者调用一些class，例如Hibernate 等，在这种时候需要设置一个比较大的持久代空间来存放这些运行过程中新增的类。持久代也称方法区，具体的回收可参见上文2.5节。
## 垃圾收集器
![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/JVM/GC/jvm-gc04.png)
**Serial收集器（复制算法)**
新生代单线程收集器，标记和清理都是单线程，优点是简单高效。是client级别默认的GC方式，可以通过-XX:+UseSerialGC来强制指定。
**Serial Old收集器(标记-整理算法)**
老年代单线程收集器，Serial收集器的老年代版本。
**ParNew收集器(停止-复制算法)　**
新生代收集器，可以认为是Serial收集器的多线程版本,在多核CPU环境下有着比Serial更好的表现。
**Parallel Scavenge收集器(停止-复制算法)**
并行收集器，追求高吞吐量，高效利用CPU。吞吐量一般为99%， 吞吐量= 用户线程时间/(用户线程时间+GC线程时间)。适合后台应用等对交互相应要求不高的场景。是server级别默认采用的GC方式，可用-XX:+UseParallelGC来强制指定，用-XX:ParallelGCThreads=4来指定线程数。
**Parallel Old收集器(停止-复制算法)**
Parallel Scavenge收集器的老年代版本，并行收集器，吞吐量优先。
**CMS(Concurrent Mark Sweep)收集器（标记-清理算法）**
高并发、低停顿，追求最短GC回收停顿时间，cpu占用比较高，响应时间快，停顿时间短，多核cpu 追求高响应时间的选择。
**Z收集器(ZGC)（标记-清理算法）**
ZGC主要新增了两项技术，一个是着色指针Colored Pointer，另一个是读屏障Load Barrier。
ZGC 是一个并发、基于区域（region）、增量式压缩的收集器。Stop-The-World 阶段只会在根对象扫描（root scanning）阶段发生，这样的话 GC 暂停时间并不会随着堆和存活对象的数量而增加。
## GC参数

**GC日志参数**：

| -XX:+PrintGC                         | 打印GC日志，和 -verbose:gc 是相同的命令              |
| :----------------------------------: | :--------------------------------------------------: |
| -XX:+PrintGCDetails                  | 打印GC的详细日志                                     |
| -XX:+PrintGCTimeStamps               | 打印GC的时间戳（JVM启动到GC发生所经历的时间）        |
| -XX:+PrintGCDateStamps               | 打印GC的日期时间（如：2019-05-06T19:34:52.072+0800） |
| -XX:+PrintHeapAtGC                   | 打印GC前后的详细的堆信息                             |
| -Xloggc:logs/gc.log.`date +%Y-%m-%d` | GC日志输出到指定文件                                 |

**垃圾收集常用参数**：

| UseSerialGC                        | 虚拟机运行在Client 模式下的默认值，打开此开关后，使用Serial + Serial Old 的收集器组合进行内存回收                                                        |
| :--------------------------------: | :------------------------------------------------------------------------------------------------------------------------------------------------------: |
| UseParNewGC                        | 打开此开关后，使用ParNew + Serial Old 的收集器组合进行内存回收                                                                                           |
| UseConcMarkSweepGC                 | 打开此开关后，使用ParNew + CMS + Serial Old 的收集器组合进行内存回收。Serial Old 收集器将作为CMS 收集器出现Concurrent Mode Failure失败后的后备收集器使用 |
| UseParallelGC                      | 虚拟机运行在Server 模式下的默认值，打开此开关后，使用ParallelScavenge + Serial Old（PS MarkSweep）的收集器组合进行内存回收                               |
| UseParallelOldGC                   | 打开此开关后，使用Parallel Scavenge + Parallel Old 的收集器组合进行内存回收                                                                              |
| SurvivorRatio                      | 新生代中Eden 区域与Survivor 区域的容量比值， 默认为8， 代表Eden ：Survivor=8∶1                                                                           |
| PretenureSizeThreshold             | 晋升到老年代的对象年龄。每个对象在坚持过一次Minor GC 之后，年龄就加1，当超过这个参数值时就进入老年代                                                       |
| MaxTenuringThreshold               | 直接晋升到老年代的对象大小，设置这个参数后，大于这个参数的对象将直接在老年代分配                                                                         |
| UseAdaptiveSizePolicy              | 动态调整Java 堆中各个区域的大小以及进入老年代的年龄                                                                                                       |
| HandlePromotionFailure             | 是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个Eden 和Survivor 区的所有对象都存活的极端情况                                               |
| GCTimeRatio                        | 设置并行GC 时进行内存回收的线程数                                                                                                                        |
| ParallelGCThreads                  | GC 时间占总时间的比率，默认值为99，即允许1% 的GC 时间。仅在                                                                                              |
| 使用Parallel Scavenge 收集器时生效 |
| MaxGCPauseMillis                   | 设置GC 的最大停顿时间。仅在使用Parallel Scavenge 收集器时生效                                                                                            |
| CMSInitiatingOccupancyFraction     | 设置CMS 收集器在老年代空间被使用多少后触发垃圾收集。默认值为68%，仅在使用CMS 收集器时生效                                                                |
| UseCMSCompactAtFullCollection      | 设置CMS 收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅                                                                                             |
| 在使用CMS 收集器时生效             |
| CMSFullGCsBeforeCompaction         | 设置CMS 收集器在进行若干次垃圾收集后再启动一次内存碎片整理。仅在使用CMS 收集器时生效                                                                     |
	