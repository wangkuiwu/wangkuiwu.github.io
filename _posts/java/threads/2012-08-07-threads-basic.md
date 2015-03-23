---
layout: post
title: "Java多线程系列--“基础篇”07之 线程休眠"
description: "java threads"
category: java
tags: [java]
date: 2012-08-07 09:01
---

> 本章，会对Thread中sleep()方法进行介绍。

> **目录**  
[1. sleep()介绍](#anchor1)   
[2. sleep()示例](#anchor2)   
[3. sleep() 与 wait()的比较](#anchor3)   


<a name="anchor1"></a>
# 1. sleep()介绍

sleep() 定义在Thread.java中。

sleep() 的作用是让当前线程休眠，即当前线程会从“运行状态”进入到“休眠(阻塞)状态”。sleep()会指定休眠时间，线程休眠的时间会大于/等于该休眠时间；在线程重新被唤醒时，它会由“阻塞状态”变成“就绪状态”，从而等待cpu的调度执行。

 
<a name="anchor2"></a>
# 2. sleep()示例

下面通过一个简单示例演示sleep()的用法。

    // SleepTest.java的源码
    class ThreadA extends Thread{
        public ThreadA(String name){ 
            super(name); 
        } 
        public synchronized void run() { 
            try {
                for(int i=0; i <10; i++){ 
                    System.out.printf("%s: %d\n", this.getName(), i); 
                    // i能被4整除时，休眠100毫秒
                    if (i%4 == 0)
                        Thread.sleep(100);
                } 
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } 
    } 

    public class SleepTest{ 
        public static void main(String[] args){ 
            ThreadA t1 = new ThreadA("t1"); 
            t1.start(); 
        } 
    } 

运行结果：

    t1: 0
    t1: 1
    t1: 2
    t1: 3
    t1: 4
    t1: 5
    t1: 6
    t1: 7
    t1: 8
    t1: 9

结果说明：  
程序比较简单，在主线程main中启动线程t1。t1启动之后，当t1中的计算i能被4整除时，t1会通过Thread.sleep(100)休眠100毫秒。

 
<a name="anchor3"></a>
# 3. sleep() 与 wait()的比较

我们知道，wait()的作用是让当前线程由“运行状态”进入“等待(阻塞)状态”的同时，也会释放同步锁。而sleep()的作用是也是让当前线程由“运行状态”进入到“休眠(阻塞)状态”。  
但是，wait()会释放对象的同步锁，而sleep()则不会释放锁。

下面通过示例演示sleep()是不会释放锁的。

    // SleepLockTest.java的源码
    public class SleepLockTest{ 

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
                    try {
                        for(int i=0; i <10; i++){ 
                            System.out.printf("%s: %d\n", this.getName(), i); 
                            // i能被4整除时，休眠100毫秒
                            if (i%4 == 0)
                                Thread.sleep(100);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            } 
        } 
    } 

运行结果：

    t1: 0
    t1: 1
    t1: 2
    t1: 3
    t1: 4
    t1: 5
    t1: 6
    t1: 7
    t1: 8
    t1: 9
    t2: 0
    t2: 1
    t2: 2
    t2: 3
    t2: 4
    t2: 5
    t2: 6
    t2: 7
    t2: 8
    t2: 9

结果说明：  
主线程main中启动了两个线程t1和t2。t1和t2在run()会引用同一个对象的同步锁，即synchronized(obj)。在t1运行过程中，虽然它会调用Thread.sleep(100)；但是，t2是不会获取cpu执行权的。因为，t1并没有释放“obj所持有的同步锁”！

注意，若我们注释掉synchronized (obj)后再次执行该程序，t1和t2是可以相互切换的。下面是注释调synchronized(obj) 之后的源码：

    // SleepLockTest.java的源码(注释掉synchronized(obj))
    public class SleepLockTest{ 

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
    //            synchronized (obj) {
                    try {
                        for(int i=0; i <10; i++){ 
                            System.out.printf("%s: %d\n", this.getName(), i); 
                            // i能被4整除时，休眠100毫秒
                            if (i%4 == 0)
                                Thread.sleep(100);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
    //            }
            } 
        } 
    } 

