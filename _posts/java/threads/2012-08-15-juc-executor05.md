---
layout: post
title: "Java多线程系列--“JUC线程池”05之 线程池原理(四)"
description: "java threads"
category: java
tags: [java]
date: 2012-08-15 09:05
---
 
> 本章介绍线程池的拒绝策略。

> **目录**  
[1. 拒绝策略介绍](#anchor1)  
[2. 拒绝策略对比和示例](#anchor2)  


 
 
<a name="anchor1"></a>
# 1. 拒绝策略介绍

线程池的拒绝策略，是指当任务添加到线程池中被拒绝，而采取的处理措施。

当任务添加到线程池中之所以被拒绝，可能是由于：  
第一，线程池异常关闭。  
第二，任务数量超过线程池的最大限制。

线程池共包括4种拒绝策略，它们分别是：**AbortPolicy, CallerRunsPolicy, DiscardOldestPolicy和DiscardPolicy**。

|   策略              |             说明                |
| ------------------- | ------------------------------- |
| AbortPolicy         | 当任务添加到线程池中被拒绝时，它将抛出 RejectedExecutionException 异常 |
| CallerRunsPolicy    | 当任务添加到线程池中被拒绝时，会在线程池当前正在运行的Thread线程池中处理被拒绝的任务 |
| DiscardOldestPolicy | 当任务添加到线程池中被拒绝时，线程池会放弃等待队列中最旧的未处理任务，然后将被拒绝的任务添加到等待队列中 |
| DiscardPolicy       | 当任务添加到线程池中被拒绝时，线程池将丢弃被拒绝的任务 |

线程池默认的处理策略是**AbortPolicy**！

 
<a name="anchor2"></a>
# 2. 拒绝策略对比和示例

下面通过示例，分别演示线程池的4种拒绝策略。  
(01) DiscardPolicy 示例  
(02) DiscardOldestPolicy 示例  
(03) AbortPolicy 示例  
(04) CallerRunsPolicy 示例

<a name="anchor2_1"></a>
## 2.1 DiscardPolicy 示例

    import java.lang.reflect.Field;
    import java.util.concurrent.ArrayBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.ThreadPoolExecutor.DiscardPolicy;

    public class DiscardPolicyDemo {

        private static final int THREADS_SIZE = 1;
        private static final int CAPACITY = 1;

        public static void main(String[] args) throws Exception {

            // 创建线程池。线程池的"最大池大小"和"核心池大小"都为1(THREADS_SIZE)，"线程池"的阻塞队列容量为1(CAPACITY)。
            ThreadPoolExecutor pool = new ThreadPoolExecutor(THREADS_SIZE, THREADS_SIZE, 0, TimeUnit.SECONDS,
                    new ArrayBlockingQueue<Runnable>(CAPACITY));
            // 设置线程池的拒绝策略为"丢弃"
            pool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());

            // 新建10个任务，并将它们添加到线程池中。
            for (int i = 0; i < 10; i++) {
                Runnable myrun = new MyRunnable("task-"+i);
                pool.execute(myrun);
            }
            // 关闭线程池
            pool.shutdown();
        }
    }

    class MyRunnable implements Runnable {
        private String name;
        public MyRunnable(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            try {
                System.out.println(this.name + " is running.");
                Thread.sleep(100);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    task-0 is running.
    task-1 is running.

结果说明：线程池pool的"最大池大小"和"核心池大小"都为1(THREADS_SIZE)，这意味着"线程池能同时运行的任务数量最大只能是1"。  
线程池pool的阻塞队列是ArrayBlockingQueue，ArrayBlockingQueue是一个有界的阻塞队列，ArrayBlockingQueue的容量为1。这也意味着线程池的阻塞队列只能有一个线程池阻塞等待。  
根据""中分析的execute()代码可知：线程池中共运行了2个任务。第1个任务直接放到Worker中，通过线程去执行；第2个任务放到阻塞队列中等待。其他的任务都被丢弃了！

 

<a name="anchor2_2"></a>
## 2.2 DiscardOldestPolicy 示例

    import java.lang.reflect.Field;
    import java.util.concurrent.ArrayBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.ThreadPoolExecutor.DiscardOldestPolicy;

    public class DiscardOldestPolicyDemo {

        private static final int THREADS_SIZE = 1;
        private static final int CAPACITY = 1;

        public static void main(String[] args) throws Exception {

            // 创建线程池。线程池的"最大池大小"和"核心池大小"都为1(THREADS_SIZE)，"线程池"的阻塞队列容量为1(CAPACITY)。
            ThreadPoolExecutor pool = new ThreadPoolExecutor(THREADS_SIZE, THREADS_SIZE, 0, TimeUnit.SECONDS,
                    new ArrayBlockingQueue<Runnable>(CAPACITY));
            // 设置线程池的拒绝策略为"DiscardOldestPolicy"
            pool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardOldestPolicy());

            // 新建10个任务，并将它们添加到线程池中。
            for (int i = 0; i < 10; i++) {
                Runnable myrun = new MyRunnable("task-"+i);
                pool.execute(myrun);
            }
            // 关闭线程池
            pool.shutdown();
        }
    }

    class MyRunnable implements Runnable {
        private String name;
        public MyRunnable(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            try {
                System.out.println(this.name + " is running.");
                Thread.sleep(200);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    task-0 is running.
    task-9 is running.

结果说明：将"线程池的拒绝策略"由DiscardPolicy修改为DiscardOldestPolicy之后，当有任务添加到线程池被拒绝时，线程池会丢弃阻塞队列中末尾的任务，然后将被拒绝的任务添加到末尾。

 

<a name="anchor2_3"></a>
## 2.3 AbortPolicy 示例

    import java.lang.reflect.Field;
    import java.util.concurrent.ArrayBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.ThreadPoolExecutor.AbortPolicy;
    import java.util.concurrent.RejectedExecutionException;

    public class AbortPolicyDemo {

        private static final int THREADS_SIZE = 1;
        private static final int CAPACITY = 1;

        public static void main(String[] args) throws Exception {

            // 创建线程池。线程池的"最大池大小"和"核心池大小"都为1(THREADS_SIZE)，"线程池"的阻塞队列容量为1(CAPACITY)。
            ThreadPoolExecutor pool = new ThreadPoolExecutor(THREADS_SIZE, THREADS_SIZE, 0, TimeUnit.SECONDS,
                    new ArrayBlockingQueue<Runnable>(CAPACITY));
            // 设置线程池的拒绝策略为"抛出异常"
            pool.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());

            try {

                // 新建10个任务，并将它们添加到线程池中。
                for (int i = 0; i < 10; i++) {
                    Runnable myrun = new MyRunnable("task-"+i);
                    pool.execute(myrun);
                }
            } catch (RejectedExecutionException e) {
                e.printStackTrace();
                // 关闭线程池
                pool.shutdown();
            }
        }
    }

    class MyRunnable implements Runnable {
        private String name;
        public MyRunnable(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            try {
                System.out.println(this.name + " is running.");
                Thread.sleep(200);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

(某一次)运行结果：

    java.util.concurrent.RejectedExecutionException
        at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:1774)
        at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:768)
        at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:656)
        at AbortPolicyDemo.main(AbortPolicyDemo.java:27)
    task-0 is running.
    task-1 is running.

结果说明：将"线程池的拒绝策略"由DiscardPolicy修改为AbortPolicy之后，当有任务添加到线程池被拒绝时，会抛出RejectedExecutionException。

 

<a name="anchor2_4"></a>
## 2.4 CallerRunsPolicy 示例

    import java.lang.reflect.Field;
    import java.util.concurrent.ArrayBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    import java.util.concurrent.ThreadPoolExecutor.CallerRunsPolicy;

    public class CallerRunsPolicyDemo {

        private static final int THREADS_SIZE = 1;
        private static final int CAPACITY = 1;

        public static void main(String[] args) throws Exception {

            // 创建线程池。线程池的"最大池大小"和"核心池大小"都为1(THREADS_SIZE)，"线程池"的阻塞队列容量为1(CAPACITY)。
            ThreadPoolExecutor pool = new ThreadPoolExecutor(THREADS_SIZE, THREADS_SIZE, 0, TimeUnit.SECONDS,
                    new ArrayBlockingQueue<Runnable>(CAPACITY));
            // 设置线程池的拒绝策略为"CallerRunsPolicy"
            pool.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

            // 新建10个任务，并将它们添加到线程池中。
            for (int i = 0; i < 10; i++) {
                Runnable myrun = new MyRunnable("task-"+i);
                pool.execute(myrun);
            }

            // 关闭线程池
            pool.shutdown();
        }
    }

    class MyRunnable implements Runnable {
        private String name;
        public MyRunnable(String name) {
            this.name = name;
        }
        @Override
        public void run() {
            try {
                System.out.println(this.name + " is running.");
                Thread.sleep(100);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

(某一次)运行结果：

    task-2 is running.
    task-3 is running.
    task-4 is running.
    task-5 is running.
    task-6 is running.
    task-7 is running.
    task-8 is running.
    task-9 is running.
    task-0 is running.
    task-1 is running.

结果说明：将"线程池的拒绝策略"由DiscardPolicy修改为CallerRunsPolicy之后，当有任务添加到线程池被拒绝时，线程池会将被拒绝的任务添加到"线程池正在运行的线程"中取运行。

 
