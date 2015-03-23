---
layout: post
title: "Java多线程系列--“基础篇”08之 join()"
description: "java threads"
category: java
tags: [java]
date: 2012-08-08 09:01
---

> 本章，会对Thread中join()方法进行介绍。

> **目录**  
[1. join()介绍](#anchor1)   
[2. join()源码分析(基于JDK1.7.0_40)](#anchor2)   
[3. join()示例](#anchor3)   


<a name="anchor1"></a>
# 1. join()介绍

join() 定义在Thread.java中。

join() 的作用：让“主线程”等待“子线程”结束之后才能继续运行。这句话可能有点晦涩，我们还是通过例子去理解：

    // 主线程
    public class Father extends Thread {
        public void run() {
            Son s = new Son();
            s.start();
            s.join();
            ...
        }
    }
    // 子线程
    public class Son extends Thread {
        public void run() {
            ...
        }
    }

说明：  
上面的有两个类Father(主线程类)和Son(子线程类)。因为Son是在Father中创建并启动的，所以，Father是主线程类，Son是子线程类。  
在Father主线程中，通过new Son()新建“子线程s”。接着通过s.start()启动“子线程s”，并且调用s.join()。在调用s.join()之后，Father主线程会一直等待，直到“子线程s”运行完毕；在“子线程s”运行完毕之后，Father主线程才能接着运行。 这也就是我们所说的“join()的作用，是让主线程会等待子线程结束之后才能继续运行”！

 
<a name="anchor2"></a>
# 2. join()源码分析(基于JDK1.7.0_40)

    public final void join() throws InterruptedException {
        join(0);
    }

    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }

说明：  
从代码中，我们可以发现。当millis==0时，会进入while(isAlive())循环；即只要子线程是活的，主线程就不停的等待。  
我们根据上面解释join()作用时的代码来理解join()的用法！

**问题**：虽然s.join()被调用的地方是发生在“Father主线程”中，但是s.join()是通过“子线程s”去调用的join()。那么，join()方法中的isAlive()应该是判断“子线程s”是不是Alive状态；对应的wait(0)也应该是“让子线程s”等待才对。但如果是这样的话，s.join()的作用怎么可能是“让主线程等待，直到子线程s完成为止”呢，应该是让"子线程等待才对(因为调用子线程对象s的wait方法嘛)"？

**答案**：wait()的作用是让“当前线程”等待，而这里的“当前线程”是指当前在CPU上运行的线程。所以，虽然是调用子线程的wait()方法，但是它是通过“主线程”去调用的；所以，休眠的是主线程，而不是“子线程”！

 
<a name="anchor3"></a>
# 3. join()示例

在理解join()的作用之后，接下来通过示例查看join()的用法。

    // JoinTest.java的源码
    public class JoinTest{

        public static void main(String[] args){ 
            try {
                ThreadA t1 = new ThreadA("t1"); // 新建“线程t1”

                t1.start();                     // 启动“线程t1”
                t1.join();                        // 将“线程t1”加入到“主线程main”中，并且“主线程main()会等待它的完成”
                System.out.printf("%s finish\n", Thread.currentThread().getName()); 
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } 

        static class ThreadA extends Thread{

            public ThreadA(String name){ 
                super(name); 
            } 
            public void run(){ 
                System.out.printf("%s start\n", this.getName()); 

                // 延时操作
                for(int i=0; i <1000000; i++)
                   ;

                System.out.printf("%s finish\n", this.getName()); 
            } 
        } 
    }

运行结果：

    t1 start
    t1 finish
    main finish

结果说明：  
运行流程如图   
(01) 在“主线程main”中通过 new ThreadA("t1") 新建“线程t1”。 接着，通过 t1.start() 启动“线程t1”，并执行t1.join()。  
(02) 执行t1.join()之后，“主线程main”会进入“阻塞状态”等待t1运行结束。“子线程t1”结束之后，会唤醒“主线程main”，“主线程”重新获取cpu执行权，继续运行。  

![img](/media/pic/java/threads/basic08.png)
 

