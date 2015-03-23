---
layout: post
title: "Java多线程系列--“JUC线程池”03之 线程池原理(二)"
description: "java threads"
category: java
tags: [java]
date: 2012-08-15 09:03
---
 
> 在前面一章"Java多线程系列--“JUC线程池”02之 线程池原理(一)"中介绍了线程池的数据结构，本章会通过分析线程池的源码，对线程池进行说明。

> **目录**  
[1. 线程池示例](#anchor1)  
[2. 线程池源码分析](#anchor2)  


 
<a name="anchor1"></a>
# 1. 线程池示例

在分析线程池之前，先看一个简单的线程池示例。

    import java.util.concurrent.Executors;
    import java.util.concurrent.ExecutorService;

    public class ThreadPoolDemo1 {

        public static void main(String[] args) {
            // 创建一个可重用固定线程数的线程池
            ExecutorService pool = Executors.newFixedThreadPool(2);
            // 创建实现了Runnable接口对象，Thread对象当然也实现了Runnable接口
            Thread ta = new MyThread();
            Thread tb = new MyThread();
            Thread tc = new MyThread();
            Thread td = new MyThread();
            Thread te = new MyThread();
            // 将线程放入池中进行执行
            pool.execute(ta);
            pool.execute(tb);
            pool.execute(tc);
            pool.execute(td);
            pool.execute(te);
            // 关闭线程池
            pool.shutdown();
        }
    }

    class MyThread extends Thread {

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+ " is running.");
        }
    }

运行结果：

    pool-1-thread-1 is running.
    pool-1-thread-2 is running.
    pool-1-thread-1 is running.
    pool-1-thread-2 is running.
    pool-1-thread-1 is running.

示例中，包括了线程池的创建，将任务添加到线程池中，关闭线程池这3个主要的步骤。稍后，我们会从这3个方面来分析ThreadPoolExecutor。


 
<a name="anchor2"></a>
# 2. 线程池源码分析

## (一) 创建“线程池”

下面以newFixedThreadPool()介绍线程池的创建过程。

### 1. newFixedThreadPool()

newFixedThreadPool()在Executors.java中定义，源码如下：

    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

说明：newFixedThreadPool(int nThreads)的作用是创建一个线程池，线程池的容量是nThreads。  
newFixedThreadPool()在调用ThreadPoolExecutor()时，会传递一个LinkedBlockingQueue()对象，而LinkedBlockingQueue是单向链表实现的阻塞队列。在线程池中，就是通过该阻塞队列来实现"当线程池中任务数量超过允许的任务数量时，部分任务会阻塞等待"。  
关于LinkedBlockingQueue的实现细节，读者可以参考"Java多线程系列--“JUC集合”08之 LinkedBlockingQueue"。

 

### 2. ThreadPoolExecutor()

ThreadPoolExecutor()在ThreadPoolExecutor.java中定义，源码如下：

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

说明：该函数实际上是调用ThreadPoolExecutor的另外一个构造函数。该函数的源码如下：

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        // 核心池大小
        this.corePoolSize = corePoolSize;
        // 最大池大小
        this.maximumPoolSize = maximumPoolSize;
        // 线程池的等待队列
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        // 线程工厂对象
        this.threadFactory = threadFactory;
        // 拒绝策略的句柄
        this.handler = handler;
    }

说明：在ThreadPoolExecutor()的构造函数中，进行的是初始化工作。  
corePoolSize, maximumPoolSize, unit, keepAliveTime和workQueue这些变量的值是已知的，它们都是通过newFixedThreadPool()传递而来。下面看看threadFactory和handler对象。

 

**2.1 ThreadFactory**

线程池中的ThreadFactory是一个线程工厂，线程池创建线程都是通过线程工厂对象(threadFactory)来完成的。

上面所说的threadFactory对象，是通过 Executors.defaultThreadFactory()返回的。Executors.java中的defaultThreadFactory()源码如下：

    public static ThreadFactory defaultThreadFactory() {
        return new DefaultThreadFactory();
    }

defaultThreadFactory()返回DefaultThreadFactory对象。Executors.java中的DefaultThreadFactory()源码如下：

 
    static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        // 提供创建线程的API。
        public Thread newThread(Runnable r) {
            // 线程对应的任务是Runnable对象r
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            // 设为“非守护线程”
            if (t.isDaemon())
                t.setDaemon(false);
            // 将优先级设为“Thread.NORM_PRIORITY”
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }

 

说明：ThreadFactory的作用就是提供创建线程的功能的线程工厂。  
它是通过newThread()提供创建线程功能的，下面简单说说newThread()。newThread()创建的线程对应的任务是Runnable对象，它创建的线程都是“非守护线程”而且“线程优先级都是Thread.NORM_PRIORITY”。

 

**2.2 RejectedExecutionHandler**

handler是ThreadPoolExecutor中拒绝策略的处理句柄。所谓拒绝策略，是指将任务添加到线程池中时，线程池拒绝该任务所采取的相应策略。

线程池默认会采用的是defaultHandler策略，即AbortPolicy策略。在AbortPolicy策略中，线程池拒绝任务时会抛出异常！  
defaultHandler的定义如下：

    private static final RejectedExecutionHandler defaultHandler = new AbortPolicy();

AbortPolicy的源码如下：

    public static class AbortPolicy implements RejectedExecutionHandler {
        public AbortPolicy() { }

        // 抛出异常
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }

 

## (二) 添加任务到“线程池”

### 1. execute()

execute()定义在ThreadPoolExecutor.java中，源码如下：

    public void execute(Runnable command) {
        // 如果任务为null，则抛出异常。
        if (command == null)
            throw new NullPointerException();
        // 获取ctl对应的int值。该int值保存了"线程池中任务的数量"和"线程池状态"信息
        int c = ctl.get();
        // 当线程池中的任务数量 < "核心池大小"时，即线程池中少于corePoolSize个任务。
        // 则通过addWorker(command, true)新建一个线程，并将任务(command)添加到该线程中；然后，启动该线程从而执行任务。
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 当线程池中的任务数量 >= "核心池大小"时，
        // 而且，"线程池处于允许状态"时，则尝试将任务添加到阻塞队列中。
        if (isRunning(c) && workQueue.offer(command)) {
            // 再次确认“线程池状态”，若线程池异常终止了，则删除任务；然后通过reject()执行相应的拒绝策略的内容。
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 否则，如果"线程池中任务数量"为0，则通过addWorker(null, false)尝试新建一个线程，新建线程对应的任务为null。
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 通过addWorker(command, false)新建一个线程，并将任务(command)添加到该线程中；然后，启动该线程从而执行任务。
        // 如果addWorker(command, false)执行失败，则通过reject()执行相应的拒绝策略的内容。
        else if (!addWorker(command, false))
            reject(command);
    }

说明：execute()的作用是将任务添加到线程池中执行。它会分为3种情况进行处理：  
&nbsp;&nbsp;&nbsp;&nbsp; 情况1 -- 如果"线程池中任务数量" < "核心池大小"时，即线程池中少于corePoolSize个任务；此时就新建一个线程，并将该任务添加到线程中进行执行。  
&nbsp;&nbsp;&nbsp;&nbsp; 情况2 -- 如果"线程池中任务数量" >= "核心池大小"，并且"线程池是允许状态"；此时，则将任务添加到阻塞队列中阻塞等待。在该情况下，会再次确认"线程池的状态"，如果"第2次读到的线程池状态"和"第1次读到的线程池状态"不同，则从阻塞队列中删除该任务。  
&nbsp;&nbsp;&nbsp;&nbsp; 情况3 -- 非以上两种情况。在这种情况下，尝试新建一个线程，并将该任务添加到线程中进行执行。如果执行失败，则通过reject()拒绝该任务。

 

### 2. addWorker()

addWorker()的源码如下：

    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        // 更新"线程池状态和计数"标记，即更新ctl。
        for (;;) {
            // 获取ctl对应的int值。该int值保存了"线程池中任务的数量"和"线程池状态"信息
            int c = ctl.get();
            // 获取线程池状态。
            int rs = runStateOf(c);

            // 有效性检查
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                // 获取线程池中任务的数量。
                int wc = workerCountOf(c);
                // 如果"线程池中任务的数量"超过限制，则返回false。
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 通过CAS函数将c的值+1。操作失败的话，则退出循环。
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 检查"线程池状态"，如果与之前的状态不同，则从retry重新开始。
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        // 添加任务到线程池，并启动任务所在的线程。
        try {
            final ReentrantLock mainLock = this.mainLock;
            // 新建Worker，并且指定firstTask为Worker的第一个任务。
            w = new Worker(firstTask);
            // 获取Worker对应的线程。
            final Thread t = w.thread;
            if (t != null) {
                // 获取锁
                mainLock.lock();
                try {
                    int c = ctl.get();
                    int rs = runStateOf(c);

                    // 再次确认"线程池状态"
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 将Worker对象(w)添加到"线程池的Worker集合(workers)"中
                        workers.add(w);
                        // 更新largestPoolSize
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    // 释放锁
                    mainLock.unlock();
                }
                // 如果"成功将任务添加到线程池"中，则启动任务所在的线程。 
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        // 返回任务是否启动。
        return workerStarted;
    }

说明：  
addWorker(Runnable firstTask, boolean core) 的作用是将任务(firstTask)添加到线程池中，并启动该任务。  
core为true的话，则以corePoolSize为界限，若"线程池中已有任务数量>=corePoolSize"，则返回false；core为false的话，则以maximumPoolSize为界限，若"线程池中已有任务数量>=maximumPoolSize"，则返回false。  
addWorker()会先通过for循环不断尝试更新ctl状态，ctl记录了"线程池中任务数量和线程池状态"。  
更新成功之后，再通过try模块来将任务添加到线程池中，并启动任务所在的线程。

从addWorker()中，我们能清晰的发现：线程池在添加任务时，会创建任务对应的Worker对象；而一个Workder对象包含一个Thread对象。(01) 通过将Worker对象添加到"线程的workers集合"中，从而实现将任务添加到线程池中。 (02) 通过启动Worker对应的Thread线程，则执行该任务。

 

### 3. submit()

补充说明一点，submit()实际上也是通过调用execute()实现的，源码如下：

    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

 

## (三) 关闭“线程池”

shutdown()的源码如下：

    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        // 获取锁
        mainLock.lock();
        try {
            // 检查终止线程池的“线程”是否有权限。
            checkShutdownAccess();
            // 设置线程池的状态为关闭状态。
            advanceRunState(SHUTDOWN);
            // 中断线程池中空闲的线程。
            interruptIdleWorkers();
            // 钩子函数，在ThreadPoolExecutor中没有任何动作。
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            // 释放锁
            mainLock.unlock();
        }
        // 尝试终止线程池
        tryTerminate();
    }

说明：shutdown()的作用是关闭线程池。

