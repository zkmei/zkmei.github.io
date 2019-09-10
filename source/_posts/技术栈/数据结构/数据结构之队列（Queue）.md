---
title: 数据结构之队列（Queue）
date: 2019-07-06 00:30:00
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Queue/java_queue_01.png
categories: 技术栈
tags:
  - 数据结构
---
# 数据结构之队列——Queue详细分析
## 概念：
Queue： 基本上，一个队列就是一个先入先出（FIFO）的数据结构
Queue接口与List、Set同一级别，都是继承了Collection接口。LinkedList实现了Deque接口。

## Queue的介绍
![Java Queue](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Queue/java_queue_01.png)
1. 没有实现的阻塞接口的LinkedList： 实现了java.util.Queue接口和java.util.AbstractQueue接口
　　内置的不阻塞队列： PriorityQueue 和 ConcurrentLinkedQueue
　　PriorityQueue 和 ConcurrentLinkedQueue 类在 Collection Framework 中加入两个具体集合实现。 
　　PriorityQueue 类实质上维护了一个有序列表。加入到 Queue 中的元素根据它们的天然排序（通过其 java.util.Comparable 实现）或者根据传递给构造函数的 java.util.Comparator 实现来定位。
　　ConcurrentLinkedQueue 是基于链接节点的、线程安全的队列。并发访问不需要同步。因为它在队列的尾部添加元素并从头部删除它们，所以只要不需要知道队列的大 小，　　　　    　　ConcurrentLinkedQueue 对公共集合的共享访问就可以工作得很好。收集关于队列大小的信息会很慢，需要遍历队列。
2. 实现阻塞接口的：
　　java.util.concurrent 中加入了 BlockingQueue 接口和五个阻塞队列类。它实质上就是一种带有一点扭曲的 FIFO 数据结构。不是立即从队列中添加或者删除元素，线程执行操作阻塞，直到有空间或者元素可用。
五个队列所提供的各有不同：
　　ArrayBlockingQueue：一个由数组支持的有界队列。
　　LinkedBlockingQueue ：一个由链接节点支持的可选有界队列。
　　PriorityBlockingQueue ：一个由优先级堆支持的无界优先级队列。
　　DelayQueue ：一个由优先级堆支持的、基于时间的调度队列。
　　SynchronousQueue ：一个利用 BlockingQueue 接口的简单聚集（rendezvous）机制。
![Java Queue extend Collection](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Queue/java_queue_02.png)
![Java Queue extends](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Queue/java_queue_03.png)
### 阻塞队列
```java
/**
 * 阻塞队列 ArrayBlockingQueue
 */
public class BlockingQueue {
    private int queueSize = 10;
    private ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<>(queueSize);
    public static void main(String[] args)  {
        BlockingQueue test = new BlockingQueue();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
        producer.start();
        consumer.start();
    }

    class Consumer extends Thread{
        @Override
        public void run() {
            consume();
        }
        private void consume() {
            while(true){
                try {
                    queue.take();
                    System.out.println("从队列取走一个元素，队列剩余"+ queue.size() +"个元素");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    class Producer extends Thread{
        @Override
        public void run() {
            produce();
        }
        private void produce() {
            while(true){
                try {
                    queue.put(1);
                    System.out.println("向队列取中插入一个元素，队列剩余空间："+ (queueSize-queue.size()));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
### 非阻塞队列
```java
/**
 * @author MZK
 *
 * 非阻塞队列 PriorityQueue
 */
public class NoBlockingQueue {
    private int queueSize = 10;
    private PriorityQueue<Integer> queue = new PriorityQueue<Integer>(queueSize);
    public static void main(String[] args)  {
        NoBlockingQueue test = new NoBlockingQueue  ();
        Producer producer = test.new Producer();
        Consumer consumer = test.new Consumer();
        producer.start();
        consumer.start();
    }

    class Consumer extends Thread{
        @Override
        public void run() {
            consume();
        }
        private void consume() {
            while(true){
                synchronized (queue) {
                    while(queue.size() == 0){
                        try {
                            System.out.println("队列空，等待数据");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.poll();          //每次移走队首元素
                    queue.notify();
                    System.out.println("从队列取走一个元素，队列剩余"+ queue.size()+"个元素");
                }
            }
        }
    }

    class Producer extends Thread{
        @Override
        public void run() {
            produce();
        }
        private void produce() {
            while(true){
                synchronized (queue) {
                    while(queue.size() == queueSize){
                        try {
                            System.out.println("队列满，等待有空余空间");
                            queue.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                            queue.notify();
                        }
                    }
                    queue.offer(1);        //每次插入一个元素
                    queue.notify();
                    System.out.println("向队列取中插入一个元素，队列剩余空间："+ (queueSize-queue.size()));
                }
            }
        }
    }
}
```

