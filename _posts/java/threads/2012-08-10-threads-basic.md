---
layout: post
title: "Java多线程系列--“基础篇”10之 线程优先级和守护线程"
description: "java threads"
category: java
tags: [java]
date: 2012-08-10 09:01
---
 
> 本章，会对守护线程和线程优先级进行介绍。

> **目录**  
[1. 线程优先级的介绍](#anchor1)  
[2. 线程优先级的示例](#anchor2)  
[3. 守护线程的示例](#anchor3)  


 
<a name="anchor1"></a>
# 1. 线程优先级的介绍

java 中的线程优先级的范围是1～10，默认的优先级是5。“高优先级线程”会优先于“低优先级线程”执行。

java 中有两种线程：用户线程和守护线程。可以通过isDaemon()方法来区别它们：如果返回false，则说明该线程是“用户线程”；否则就是“守护线程”。  
用户线程一般用户执行用户级任务，而守护线程也就是“后台线程”，一般用来执行后台任务。需要注意的是：Java虚拟机在“用户线程”都结束后会后退出。

JDK 中关于线程优先级和守护线程的介绍如下：

> Every thread has a priority. Threads with higher priority are executed in preference to threads with lower priority. Each thread may or may not also be marked as a daemon. When code running in some thread creates a new Thread object, the new thread has its priority initially set equal to the priority of the creating thread, and is a daemon thread if and only if the creating thread is a daemon.  
When a Java Virtual Machine starts up, there is usually a single non-daemon thread (which typically calls the method named main of some designated class). The Java Virtual Machine continues to execute threads until either of the following occurs:  
The exit method of class Runtime has been called and the security manager has permitted the exit operation to take place.  
All threads that are not daemon threads have died, either by returning from the call to the run method or by throwing an exception that propagates beyond the run method.   
Marks this thread as either a daemon thread or a user thread. The Java Virtual Machine exits when the only threads running are all daemon threads.

大致意思是：

每个线程都有一个优先级。“高优先级线程”会优先于“低优先级线程”执行。每个线程都可以被标记为一个守护进程或非守护进程。在一些运行的主线程中创建新的子线程时，子线程的优先级被设置为等于“创建它的主线程的优先级”，当且仅当“创建它的主线程是守护线程”时“子线程才会是守护线程”。

当Java虚拟机启动时，通常有一个单一的非守护线程（该线程通过是通过main()方法启动）。JVM会一直运行直到下面的任意一个条件发生，JVM就会终止运行：  
(01) 调用了exit()方法，并且exit()有权限被正常执行。  
(02) 所有的“非守护线程”都死了(即JVM中仅仅只有“守护线程”)。  

每一个线程都被标记为“守护线程”或“用户线程”。当只有守护线程运行时，JVM会自动退出。

 
<a name="anchor2"></a>
# 2. 线程优先级的示例

我们先看看优先级的示例 

    class MyThread extends Thread{
        public MyThread(String name) {
            super(name);
        }

        public void run(){
            for (int i=0; i<5; i++) {
                System.out.println(Thread.currentThread().getName()
                        +"("+Thread.currentThread().getPriority()+ ")"
                        +", loop "+i);
            }
        } 
    }; 

    public class Demo {  
        public static void main(String[] args) {  

            System.out.println(Thread.currentThread().getName()
                    +"("+Thread.currentThread().getPriority()+ ")");

            Thread t1=new MyThread("t1");    // 新建t1
            Thread t2=new MyThread("t2");    // 新建t2
            t1.setPriority(1);                // 设置t1的优先级为1
            t2.setPriority(10);                // 设置t2的优先级为10
            t1.start();                        // 启动t1
            t2.start();                        // 启动t2
        }  
    }

运行结果：

    main(5)
    t1(1), loop 0
    t2(10), loop 0
    t1(1), loop 1
    t2(10), loop 1
    t1(1), loop 2
    t2(10), loop 2
    t1(1), loop 3
    t2(10), loop 3
    t1(1), loop 4
    t2(10), loop 4

结果说明：  
(01) 主线程main的优先级是5。  
(02) t1的优先级被设为1，而t2的优先级被设为10。cpu在执行t1和t2的时候，根据时间片轮循调度，所以能够并发执行。

 
<a name="anchor3"></a>
# 3. 守护线程的示例

下面是守护线程的示例。

    // Demo.java
    class MyThread extends Thread{
        public MyThread(String name) {
            super(name);
        }

        public void run(){
            try {
                for (int i=0; i<5; i++) {
                    Thread.sleep(3);
                    System.out.println(this.getName() +"(isDaemon="+this.isDaemon()+ ")" +", loop "+i);
                }
            } catch (InterruptedException e) {
            }
        } 
    }; 

    class MyDaemon extends Thread{  
        public MyDaemon(String name) {
            super(name);
        }

        public void run(){
            try {
                for (int i=0; i<10000; i++) {
                    Thread.sleep(1);
                    System.out.println(this.getName() +"(isDaemon="+this.isDaemon()+ ")" +", loop "+i);
                }
            } catch (InterruptedException e) {
            }
        } 
    }
    public class Demo {  
        public static void main(String[] args) {  

            System.out.println(Thread.currentThread().getName()
                    +"(isDaemon="+Thread.currentThread().isDaemon()+ ")");

            Thread t1=new MyThread("t1");    // 新建t1
            Thread t2=new MyDaemon("t2");    // 新建t2
            t2.setDaemon(true);                // 设置t2为守护线程
            t1.start();                        // 启动t1
            t2.start();                        // 启动t2
        }  
    }

运行结果：

    main(isDaemon=false)
    t2(isDaemon=true), loop 0
    t2(isDaemon=true), loop 1
    t1(isDaemon=false), loop 0
    t2(isDaemon=true), loop 2
    t2(isDaemon=true), loop 3
    t1(isDaemon=false), loop 1
    t2(isDaemon=true), loop 4
    t2(isDaemon=true), loop 5
    t2(isDaemon=true), loop 6
    t1(isDaemon=false), loop 2
    t2(isDaemon=true), loop 7
    t2(isDaemon=true), loop 8
    t2(isDaemon=true), loop 9
    t1(isDaemon=false), loop 3
    t2(isDaemon=true), loop 10
    t2(isDaemon=true), loop 11
    t1(isDaemon=false), loop 4
    t2(isDaemon=true), loop 12

结果说明：  
(01) 主线程main是用户线程，它创建的子线程t1也是用户线程。  
(02) t2是守护线程。在“主线程main”和“子线程t1”(它们都是用户线程)执行完毕，只剩t2这个守护线程的时候，JVM自动退出。

