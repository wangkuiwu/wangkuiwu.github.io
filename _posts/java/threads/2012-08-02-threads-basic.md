---
layout: post
title: "Java多线程系列--“基础篇”02之 常用的实现多线程的两种方式"
description: "java threads"
category: java
tags: [java]
date: 2012-08-02 09:01
---

> 本章，我们学习“常用的实现多线程的2种方式”：Thread 和 Runnable。  
之所以说是常用的，是因为通过还可以通过java.util.concurrent包中的线程池来实现多线程。关于线程池的内容，我们以后会详细介绍；现在，先对的Thread和Runnable进行了解。

> **目录**  
> [1. Thread和Runnable简介](#anchor1)   
> [2. Thread和Runnable的异同点](#anchor2)   
> [3. Thread和Runnable的多线程示例](#anchor3)   


<a name="anchor1"></a>
# 1. Thread和Runnable简介

Runnable 是一个接口，该接口中只包含了一个run()方法。它的定义如下：

    public interface Runnable {
        public abstract void run();
    }

Runnable的作用，实现多线程。我们可以定义一个类A实现Runnable接口；然后，通过new Thread(new A())等方式新建线程。

 

Thread 是一个类。Thread本身就实现了Runnable接口。它的声明如下：

    public class Thread implements Runnable {}

Thread的作用，实现多线程。

 
<a name="anchor2"></a>
# 2. Thread和Runnable的异同点

Thread 和 Runnable 的相同点：都是“多线程的实现方式”。

Thread 和 Runnable 的不同点：  
Thread 是类，而Runnable是接口；Thread本身是实现了Runnable接口的类。我们知道“一个类只能有一个父类，但是却能实现多个接口”，因此Runnable具有更好的扩展性。  
此外，Runnable还可以用于“资源的共享”。即，多个线程都是基于某一个Runnable对象建立的，它们会共享Runnable对象上的资源。

通常，建议通过“Runnable”实现多线程！

 
<a name="anchor3"></a>
# 3. Thread和Runnable的多线程示例
## 3.1 Thread的多线程示例

下面通过示例更好的理解Thread和Runnable，借鉴网上一个例子比较具有说服性的例子。

    // ThreadTest.java 源码
    class MyThread extends Thread{
        private int ticket=10;  
        public void run(){
            for(int i=0;i<20;i++){ 
                if(this.ticket>0){
                    System.out.println(this.getName()+" 卖票：ticket"+this.ticket--);
                }
            }
        } 
    };

    public class ThreadTest {  
        public static void main(String[] args) {  
            // 启动3个线程t1,t2,t3；每个线程各卖10张票！
            MyThread t1=new MyThread();
            MyThread t2=new MyThread();
            MyThread t3=new MyThread();
            t1.start();
            t2.start();
            t3.start();
        }  
    } 

运行结果：

    Thread-0 卖票：ticket10
    Thread-1 卖票：ticket10
    Thread-2 卖票：ticket10
    Thread-1 卖票：ticket9
    Thread-0 卖票：ticket9
    Thread-1 卖票：ticket8
    Thread-2 卖票：ticket9
    Thread-1 卖票：ticket7
    Thread-0 卖票：ticket8
    Thread-1 卖票：ticket6
    Thread-2 卖票：ticket8
    Thread-1 卖票：ticket5
    Thread-0 卖票：ticket7
    Thread-1 卖票：ticket4
    Thread-2 卖票：ticket7
    Thread-1 卖票：ticket3
    Thread-0 卖票：ticket6
    Thread-1 卖票：ticket2
    Thread-2 卖票：ticket6
    Thread-2 卖票：ticket5
    Thread-2 卖票：ticket4
    Thread-1 卖票：ticket1
    Thread-0 卖票：ticket5
    Thread-2 卖票：ticket3
    Thread-0 卖票：ticket4
    Thread-2 卖票：ticket2
    Thread-0 卖票：ticket3
    Thread-2 卖票：ticket1
    Thread-0 卖票：ticket2
    Thread-0 卖票：ticket1

结果说明：  
(01) MyThread继承于Thread，它是自定义个线程。每个MyThread都会卖出10张票。  
(02) 主线程main创建并启动3个MyThread子线程。每个子线程都各自卖出了10张票。

 
## 3.2 Runnable的多线程示例

下面，我们对上面的程序进行修改。通过Runnable实现一个接口，从而实现多线程。

    // RunnableTest.java 源码
    class MyThread implements Runnable{  
        private int ticket=10;  
        public void run(){
            for(int i=0;i<20;i++){ 
                if(this.ticket>0){
                    System.out.println(Thread.currentThread().getName()+" 卖票：ticket"+this.ticket--);
                }
            }
        } 
    }; 

    public class RunnableTest {  
        public static void main(String[] args) {  
            MyThread mt=new MyThread();

            // 启动3个线程t1,t2,t3(它们共用一个Runnable对象)，这3个线程一共卖10张票！
            Thread t1=new Thread(mt);
            Thread t2=new Thread(mt);
            Thread t3=new Thread(mt);
            t1.start();
            t2.start();
            t3.start();
        }  
    }

运行结果：

    Thread-0 卖票：ticket10
    Thread-2 卖票：ticket8
    Thread-1 卖票：ticket9
    Thread-2 卖票：ticket6
    Thread-0 卖票：ticket7
    Thread-2 卖票：ticket4
    Thread-1 卖票：ticket5
    Thread-2 卖票：ticket2
    Thread-0 卖票：ticket3
    Thread-1 卖票：ticket1

结果说明：  
(01) 和上面“MyThread继承于Thread”不同；这里的MyThread实现了Thread接口。  
(02) 主线程main创建并启动3个子线程，而且这3个子线程都是基于“mt这个Runnable对象”而创建的。运行结果是这3个子线程一共卖出了10张票。这说明它们是共享了MyThread接口的。

 
