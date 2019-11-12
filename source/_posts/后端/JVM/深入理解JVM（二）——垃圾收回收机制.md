---
title: 深入理解JVM（二）——OOM和JDK工具
date: 2019-09-15 23:30:33
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/JVM/GC/jvm-gc.jpg
categories: 后端
tags:
    - JVM
    - GC
---
# 概述
## Java堆溢出
```java
/**
 * VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 * @author MZK
 */
public class HeapOOM {

	static class OOMobject{}
	
	public static void main(String[] args) {
		List<OOMobject> list = new ArrayList<>();
		
		while(true) {
			list.add(new OOMobject());
		}
	}
}
    运行结果:
    java.lang.OutOfMemoryError: Java heap space
    Dumping heap to java_pid21840.hprof ...
    Heap dump file created [28000346 bytes in 0.060 secs]
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space 
```
## 虚拟机栈和本地方法栈溢出
**单线程：**
```java
public class JavaVMStackSOF {
	private int stackLength = 1;
	public void stackLeak() {
		stackLength++;
		stackLeak();
	}

	public static void main(String[] args) {
		JavaVMStackSOF oom = new JavaVMStackSOF();
		try {
			oom.stackLeak();
		} catch (Exception e) {
			System.out.println("stack length:" + oom.stackLength);
			throw e;
		}
	}
}
    运行结果:
    Exception in thread "main" java.lang.StackOverflowError
	at com.mzk.jvm.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:8)
	at com.mzk.jvm.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:9)
	at com.mzk.jvm.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:9)
```
单线程情况下，无论由于栈帧太大还是虚拟机容量太小，内存无法分配时，虚拟机抛出的都是StackOverflowError异常。
**多线程：**
```java
/**
 * VM Args: -Xss2M
 * @author MZK
 */
public class JavaVMStackOOM {
    public JavaVMStackOOM() {
    }

    private void dontStop() {
        // $FF: Couldn't be decompiled
    }

    public void stackLeakByThread() {
        while(true) {
            Thread thread = new Thread(new Runnable() {
                public void run() {
                    JavaVMStackOOM.this.dontStop();
                }
            });
            thread.start();
        }
    }

    public static void main(String[] args) {
        JavaVMStackOOM javaVMStackOOM = new JavaVMStackOOM();
        javaVMStackOOM.stackLeakByThread();
    }
}
运行结果：
    Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
```
## 方法区和运行时常量池溢出
运行时常量池异出，在OutOfMemoryError后面跟随的提示信息是“**PermGen space**”，说明运行时常量池属于方法区（HotSpot吸泥机中的永久代）的一部分。
方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾收集器回收掉，判定条件是比较苛刻。在经常动态生成大量class的应用中，需要特别注意类的回收情况。
## 本机直接内存溢出
由DirectMemory导致的内存溢出，一个明显的特征就是在Heap Dump文件中不会看见明显的异常，如果发现OOM之后Dump文件很小，而程序使用了NIO，就有必要考虑检查一下是否是这方面原因。
## JDK工具
| 名称   | 主要作用                                                                                     |
| :----: | :------------------------------------------------------------------------------------------: |
| jps    | JVM Process Status Tool，显示知道系统内所有的HotSpot                                         |
| jstat  | JVM Statistics Monitoring Tool,用于手机 HotSpot虚拟机各方面运行数据                          |
| jinfo  | Configuration Info for Java，显示虚拟机配置信息                                              |
| jmap   | Memory Map for Java,生成虚拟机的内存转储快照（heapdump文件）                                 |
| jhat   | JVM Heap Dump Browser，用于分析heapdump文件，创建一个HTTP/HTML服务器，再浏览器上查看分析结果 |
| jstack | Stack Trace for Java，显示虚拟机的线程快照                                                   |
### jps：虚拟机进程状况工具
| 选项  | 作用                                               |
| :---: | :------------------------------------------------: |
| -q    | 只输出LVMD，省略朱磊名称                           |
| -m    | 输出虚拟机进程启动时传递给主类main()函数的参数     |
| -l    | 输出主类的全名，如果进程执行的是Jar包，输出Jar路径 |
| -v    | 输出虚拟机进程启动时JVM参数                        |
jps 命令格式
jps [ options ] [ hostid ] 
### jstat：虚拟机统计信息监视工具
| 选项             | 作用                                                                                 |
| :--------------: | :----------------------------------------------------------------------------------: |
| class            | 显示ClassLoad的相关信息；                                                            |
| compiler         | 显示JIT编译的相关信息；                                                              |
| gc               | 显示和gc相关的堆信息；                                                               |
| gccapacity       | 显示各个代的容量以及使用情况；                                                       |
| gcmetacapacity   | 显示metaspace的大小                                                                  |
| gcnew            | 显示新生代信息；                                                                     |
| gcnewcapacity    | 显示新生代大小和使用情况；                                                           |
| gcold            | 显示老年代和永久代的信息；                                                           |
| gcoldcapacity    | 显示老年代的大小；                                                                   |
| gcutil           | 显示垃圾收集信息；                                                                   |
| gccause          | 显示垃圾回收的相关信息（通-gcutil）,同时显示最后一次或当前正在发生的垃圾回收的诱因； |
| printcompilation | 输出JIT编译的方法信息；                                                              |
jstat命令格式：
jstat [ option vmid [interval[s|ms] [count]]]
例：jstat-gc 2764 250 20
选项option代表着用户徐网查询的虚拟机信息：主要分为3类：类装载、垃圾收集、运行期编译状况。
### jinfo：Java配置信息工具
jinfo命令格式：
jinfo [ option ] pid
一般使用-flag选项
### jmap：Java内存映像工具
| 选项                | 作用                                                                     |
| :-----------------: | :----------------------------------------------------------------------: |
| no option           | 查看进程的内存映像信息,类似 Solaris pmap 命令。                          |
| heap                | 显示Java堆详细信息                                                       |
| histo[:live]        | 显示堆中对象的统计信息                                                   |
| clstats             | 打印类加载器信息                                                         |
| finalizerinfo       | 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象                |
| dump:<dump-options> | 生成堆转储快照                                                           |
| F                   | 当-dump没有响应时，使用-dump或者-histo参数. 在这个模式下,live子参数无效. |
| help                | 打印帮助信息                                                             |
| J<flag>             | 指定传递给运行jmap的JVM的参数                                            |
jmap命令格式：
jmap [ option ] vmid
### jhat：虚拟机堆转储快照分析工具
JDK提供jhat命令与jmap搭配使用，来分析jmap生成堆转储快照。
内置了一个微型HTTP/HTML服务器，显示dump文件结果，但是相对来说比较简陋，其他的还有 VisualVM 、Eclipse Memory Analyzer、IBM HeapAnalyzer等工具。
### jstack：Java堆栈跟踪工具 
| 选项  | 作用                                         |
| :---: | :------------------------------------------: |
| -F    | 当正常输出的请求不被响应时，强制输出线程堆栈 |
| -l    | 出堆栈外，显示关于所的附加信息               |
| -m    | 如果调用本地方法，可现实C/C++的堆栈          |
jstack命令格式：
jstack [ option ] vmid


                                           |
	