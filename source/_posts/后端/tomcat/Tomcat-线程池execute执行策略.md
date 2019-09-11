---
title: Tomcat-线程池execute执行策略
date: 2019-07-08 19:30:33
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Thread/Thread_01.jpg
categories: 后端
tags:
    - Java
    - Tomcat
---

先说一下JDK中ThreadExecutor的execute方法的处理逻辑：

> 1.小于等于Coresize：创建线程执行； 
> 2.大于CoreSize：加入队列； 
> 3.队列满且小于maxSize：有空闲线程使用空闲线程执行，没有的话，创建线程执行；如果大于maxSize则拒绝策略执行。

&emsp;&emsp;设置的不恰当时，队列使用LinkedBlockingQueue，那么线程数量是很不可能达到maximumPoolSize，因为线程数量到了corePoolSize之后，之后新的任务是添加到队列里面去了。 
&emsp;&emsp;Tomcat中有一个StandardThreadExecutor线程池，该线程池execute执行策略是优先扩充线程到maximumPoolSize，再offer到queue，如果满了就reject。来看看StandardThreadExecutor是怎么实现的。 
&emsp;&emsp;首先Tomcat的StandardThreadExecutor类实现了org.apache.catalina.Executor接口,该接口继承了java.util.concurrent.Executor。StandardThreadExecutor内部组合了一个org.apache.tomcat.util.threads.ThreadPoolExecutor,该类继承了java.util.concurrent.ThreadPoolExecutor。
  
然后java.util.concurrent.ThreadPoolExecutor的execute方法
这是注释的翻译：
* 1.如果运行的线程小于corePoolSize，请尝试这样做用给定的命令作为第一个线程启动一个新线程任务。对addWorker的调用将自动检查runState和 workerCount，这样可以防止添加错误警报线程当它不应该时，返回false。
* 2.如果一个任务可以成功排队，那么我们仍然需要再次检查是否应该添加线程(因为上次检查后现有的已经死亡)或其他进入此方法后，池关闭。所以我们重新检查状态，如果需要，回滚排队停止，如果没有线程，则启动一个新线程。
* 3.如果无法对任务排队，则尝试添加一个新任务线程。如果它失败了，我们知道我们被关闭或饱和了所以拒绝这个任务。
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
}
```
&emsp;&emsp;addWorker是创建线程执行任务，该方法中会判断当前的线程数量是否达到maximumPoolSize；workQueue.offer是将任务添加到队列里面。 
&emsp;&emsp;org.apache.tomcat.util.threads.ThreadPoolExecutor就是对workQueue作了一些改动。StandardThreadExecutor使用的是org.apache.tomcat.util.threads.TaskQueue：
```java
public class TaskQueue extends LinkedBlockingQueue<Runnable> {

    private volatile ThreadPoolExecutor parent = null;

    public boolean force(Runnable o) {
        if ( parent==null || parent.isShutdown() ) throw new RejectedExecutionException("Executor not running, can't force a command into the queue");
        return super.offer(o); //forces the item onto the queue, to be used if the task is rejected
    }

    public boolean force(Runnable o, long timeout, TimeUnit unit) throws InterruptedException {
        if ( parent==null || parent.isShutdown() ) throw new RejectedExecutionException("Executor not running, can't force a command into the queue");
        return super.offer(o,timeout,unit); //forces the item onto the queue, to be used if the task is rejected
    }

    @Override
    public boolean offer(Runnable o) {
      //we can't do any checks
        if (parent==null) return super.offer(o);
        //we are maxed out on threads, simply queue the object
        if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);
        //we have idle threads, just add it to the queue
        if (parent.getSubmittedCount()<(parent.getPoolSize())) return super.offer(o);
        //if we have less threads than maximum force creation of a new thread
        if (parent.getPoolSize()<parent.getMaximumPoolSize()) return false;
        //if we reached here, we need to add it to the queue
        return super.offer(o);
    }

    //……
}
```
&emsp;&emsp;当parent.getPoolSize()还没达到parent.getMaximumPoolSize()大小时，返回false，这样会走addWorker分支。再来看org.apache.tomcat.util.threads.ThreadPoolExecutor的execute:
```java
public void execute(Runnable command, long timeout, TimeUnit unit) {
    submittedCount.incrementAndGet();
    try {
        super.execute(command);
    } catch (RejectedExecutionException rx) {
        if (super.getQueue() instanceof TaskQueue) {
            final TaskQueue queue = (TaskQueue)super.getQueue();
            try {
                if (!queue.force(command, timeout, unit)) {
                    submittedCount.decrementAndGet();
                    throw new RejectedExecutionException("Queue capacity is full.");
                }
            } catch (InterruptedException x) {
                submittedCount.decrementAndGet();
                throw new RejectedExecutionException(x);
            }
        } else {
            submittedCount.decrementAndGet();
            throw rx;
        }
    }
}
```
&emsp;&emsp;当抛出RejectedExecutionException时，会等待timeout后重试一次加到queue中，如果失败才最终抛出异常。 
StandardThreadExecutor指定的RejectedExecutionHandler是RejectHandler：
```java
private static class RejectHandler implements RejectedExecutionHandler {
        @Override
        public void rejectedExecution(Runnable r,
                java.util.concurrent.ThreadPoolExecutor executor) {
            throw new RejectedExecutionException();
        }
    }
```



























