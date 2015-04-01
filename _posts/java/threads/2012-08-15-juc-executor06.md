---
layout: post
title: "Java多线程系列--“JUC线程池”06之 Callable和Future"
description: "java threads"
category: java
tags: [java]
date: 2012-08-15 09:06
---
 
> 本章介绍线程池中的Callable和Future。

> **目录**  
[第1部分 Callable 和 Future 简介](#anchor1)  
[第2部分 示例和源码分析(基于JDK1.7.0_40)](#anchor2)  
 

 
<a name="anchor1"></a>
# 第1部分 Callable 和 Future 简介

Callable 和 Future 是比较有趣的一对组合。当我们需要获取线程的执行结果时，就需要用到它们。Callable用于产生结果，Future用于获取结果。

## 1. Callable

Callable 是一个接口，它只包含一个call()方法。Callable是一个返回结果并且可能抛出异常的任务。

为了便于理解，我们可以将Callable比作一个Runnable接口，而Callable的call()方法则类似于Runnable的run()方法。

Callable的源码如下：

    public interface Callable<V> {
        V call() throws Exception;
    }

说明：从中我们可以看出Callable支持泛型。

 
## 2. Future

Future 是一个接口。它用于表示异步计算的结果。提供了检查计算是否完成的方法，以等待计算的完成，并获取计算的结果。

Future的源码如下：

    public interface Future<V> {
        // 试图取消对此任务的执行。
        boolean     cancel(boolean mayInterruptIfRunning)

        // 如果在任务正常完成前将其取消，则返回 true。
        boolean     isCancelled()

        // 如果任务已完成，则返回 true。
        boolean     isDone()

        // 如有必要，等待计算完成，然后获取其结果。
        V           get() throws InterruptedException, ExecutionException;

        // 如有必要，最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）。
        V             get(long timeout, TimeUnit unit)
              throws InterruptedException, ExecutionException, TimeoutException;
    }

说明： Future用于表示异步计算的结果。它的实现类是FutureTask，在讲解FutureTask之前，我们先看看Callable, Future, FutureTask它们之间的关系图，如下：

![img](/media/pic/java/threads/juc-executor06-01.jpg)

说明：  
(01) RunnableFuture是一个接口，它继承了Runnable和Future这两个接口。RunnableFuture的源码如下：

    public interface RunnableFuture<V> extends Runnable, Future<V> {
        void run();
    }

(02) FutureTask实现了RunnableFuture接口。所以，我们也说它实现了Future接口。

 

<a name="anchor2"></a>
# 第2部分 示例和源码分析(基于JDK1.7.0_40)

我们先通过一个示例看看Callable和Future的基本用法，然后再分析示例的实现原理。

    import java.util.concurrent.Callable;
    import java.util.concurrent.Future;
    import java.util.concurrent.Executors;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.ExecutionException;

    class MyCallable implements Callable {

        @Override 
        public Integer call() throws Exception {
            int sum    = 0;
            // 执行任务
            for (int i=0; i<100; i++)
                sum += i;
            //return sum; 
            return Integer.valueOf(sum);
        } 
    }

    public class CallableTest1 {

        public static void main(String[] args) 
            throws ExecutionException, InterruptedException{
            //创建一个线程池
            ExecutorService pool = Executors.newSingleThreadExecutor();
            //创建有返回值的任务
            Callable c1 = new MyCallable();
            //执行任务并获取Future对象 
            Future f1 = pool.submit(c1);
            // 输出结果
            System.out.println(f1.get()); 
            //关闭线程池 
            pool.shutdown(); 
        }
    }

运行结果：

    4950

结果说明：在主线程main中，通过newSingleThreadExecutor()新建一个线程池。接着创建Callable对象c1，然后再通过pool.submit(c1)将c1提交到线程池中进行处理，并且将返回的结果保存到Future对象f1中。然后，我们通过f1.get()获取Callable中保存的结果；最后通过pool.shutdown()关闭线程池。

 

## 1. submit()

submit()在java/util/concurrent/AbstractExecutorService.java中实现，它的源码如下：

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        // 创建一个RunnableFuture对象
        RunnableFuture<T> ftask = newTaskFor(task);
        // 执行“任务ftask”
        execute(ftask);
        // 返回“ftask”
        return ftask;
    }

说明：submit()通过newTaskFor(task)创建了RunnableFuture对象ftask。它的源码如下：

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

 

## 2. FutureTask的构造函数

FutureTask的构造函数如下：

    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        // callable是一个Callable对象
        this.callable = callable;
        // state记录FutureTask的状态
        this.state = NEW;       // ensure visibility of callable
    }

 

## 3. FutureTask的run()方法

我们继续回到submit()的源码中。

在newTaskFor()新建一个ftask对象之后，会通过execute(ftask)执行该任务。此时ftask被当作一个Runnable对象进行执行，最终会调用到它的run()方法；ftask的run()方法在java/util/concurrent/FutureTask.java中实现，源码如下：

    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            // 将callable对象赋值给c。
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    // 执行Callable的call()方法，并保存结果到result中。
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                // 如果运行成功，则将result保存
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            // 设置“state状态标记”
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }

说明：run()中会执行Callable对象的call()方法，并且最终将结果保存到result中，并通过set(result)将result保存。  
之后调用FutureTask的get()方法，返回的就是通过set(result)保存的值。

