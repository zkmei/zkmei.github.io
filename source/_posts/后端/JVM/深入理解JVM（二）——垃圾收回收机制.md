---
title: 深入理解JVM（三）——OutOfMemoryError
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
**多线程：**
```java

```

## 方法区和运行时常量池溢出
## 本机直接内存溢出



                                           |
	