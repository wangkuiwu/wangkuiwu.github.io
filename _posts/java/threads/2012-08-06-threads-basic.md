---
layout: post
title: "Java多线程系列--“基础篇”06之 线程让步"
description: "java threads"
category: java
tags: [java]
date: 2012-08-06 09:01
---

> 本章，会对Thread中的线程让步方法yield()进行介绍。

> **目录**  
[1. yield()介绍](#anchor1)   
[2. yield()示例](#anchor2)   
[3. yield() 与 wait()的比较](#anchor3)   


<a name="anchor1"></a>
# 1. yield()介绍

yield()的作用是让步。它能让当前线程由“运行状态”进入到“就绪状态”，从而让其它具有相同优先级的等待线程获取执行权；但是，并不能保证在当前线程调用yield()之后，其它具有相同优先级的线程就一定能获得执行权；也有可能是当前线程又进入到“运行状态”继续运行！

 
<a name="anchor2"></a>
# 2. yield()示例

下面，通过示例查看它的用法。

    // YieldTest.java的源码
    class ThreadA extends Thread{
        public ThreadA(String name){ 
            super(name); 
        } 
        public synchronized void run(){ 
            for(int i=0; i <10; i++){ 
                System.out.printf("%s [%d]:%d\n", this.getName(), this.getPriority(), i); 
                // i整除4时，调用yield
                if (i%4 == 0)
                    Thread.yield();
            } 
        } 
    } 

    public class YieldTest{ 
        public static void main(String[] args){ 
            ThreadA t1 = new ThreadA("t1"); 
            ThreadA t2 = new ThreadA("t2"); 
            t1.start(); 
            t2.start();
        } 
    } 

(某一次的)运行结果:

    t1 [5]:0
    t2 [5]:0
    t1 [5]:1
    t1 [5]:2
    t1 [5]:3
    t1 [5]:4
    t1 [5]:5
    t1 [5]:6
    t1 [5]:7
    t1 [5]:8
    t1 [5]:9
    t2 [5]:1
    t2 [5]:2
    t2 [5]:3
    t2 [5]:4
    t2 [5]:5
    t2 [5]:6
    t2 [5]:7
    t2 [5]:8
    t2 [5]:9

结果说明：  
“线程t1”在能被4整数的时候，并没有切换到“线程t2”。这表明，yield()虽然可以让线程由“运行状态”进入到“就绪状态”；但是，它不一定会让其它线程获取CPU执行权(即，其它线程进入到“运行状态”)，即使这个“其它线程”与当前调用yield()的线程具有相同的优先级。

 
<a name="anchor3"></a>
# 3. yield() 与 wait()的比较

我们知道，wait()的作用是让当前线程由“运行状态”进入“等待(阻塞)状态”的同时，也会释放同步锁。而yield()的作用是让步，它也会让当前线程离开“运行状态”。它们的区别是：  
(01) wait()是让线程由“运行状态”进入到“等待(阻塞)状态”，而不yield()是让线程由“运行状态”进入到“就绪状态”。  
(02) wait()是会线程释放它所持有对象的同步锁，而yield()方法不会释放锁。

下面通过示例演示yield()是不会释放锁的。

    // YieldLockTest.java 的源码
    public class YieldLockTest{ 

        private static Object obj = new Object();

        public static void main(String[] args){ 
            ThreadA t1 = new ThreadA("t1"); 
            ThreadA t2 = new ThreadA("t2"); 
            t1.start(); 
            t2.start();
        } 

        static class ThreadA extends Thread{
            public ThreadA(String name){ 
                super(name); 
            } 
            public void run(){ 
                // 获取obj对象的同步锁
                synchronized (obj) {
                    for(int i=0; i <10; i++){ 
                        System.out.printf("%s [%d]:%d\n", this.getName(), this.getPriority(), i); 
                        // i整除4时，调用yield
                        if (i%4 == 0)
                            Thread.yield();
                    }
                }
            } 
        } 
    } 

(某一次)运行结果：

    t1 [5]:0
    t1 [5]:1
    t1 [5]:2
    t1 [5]:3
    t1 [5]:4
    t1 [5]:5
    t1 [5]:6
    t1 [5]:7
    t1 [5]:8
    t1 [5]:9
    t2 [5]:0
    t2 [5]:1
    t2 [5]:2
    t2 [5]:3
    t2 [5]:4
    t2 [5]:5
    t2 [5]:6
    t2 [5]:7
    t2 [5]:8
    t2 [5]:9

结果说明：  
主线程main中启动了两个线程t1和t2。t1和t2在run()会引用同一个对象的同步锁，即synchronized(obj)。在t1运行过程中，虽然它会调用Thread.yield()；但是，t2是不会获取cpu执行权的。因为，t1并没有释放“obj所持有的同步锁”！

 
