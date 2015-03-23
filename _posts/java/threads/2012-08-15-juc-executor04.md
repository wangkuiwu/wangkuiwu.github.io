---
layout: post
title: "Java多线程系列--“JUC线程池”04之 线程池原理(三)"
description: "java threads"
category: java
tags: [java]
date: 2012-08-15 09:04
---
 
> 本章介绍线程池的生命周期。   
在"Java多线程系列--“基础篇”01之 基本概念"中，我们介绍过，线程有5种状态：新建状态，就绪状态，运行状态，阻塞状态，死亡状态。线程池也有5种状态；然而，线程池不同于线程，线程池的5种状态是：**Running, SHUTDOWN, STOP, TIDYING, TERMINATED**。


线程池状态定义代码如下：

    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY = (1 << COUNT_BITS) - 1;

    private static final int RUNNING = -1 << COUNT_BITS;
    private static final int SHUTDOWN = 0 << COUNT_BITS;
    private static final int STOP = 1 << COUNT_BITS;
    private static final int TIDYING = 2 << COUNT_BITS;
    private static final int TERMINATED = 3 << COUNT_BITS;
    private static int ctlOf(int rs, int wc) { return rs | wc; }

说明：  
ctl是一个AtomicInteger类型的原子对象。ctl记录了"线程池中的任务数量"和"线程池状态"2个信息。  
ctl共包括32位。其中，高3位表示"线程池状态"，低29位表示"线程池中的任务数量"。


|   状态      |             说明                |
| ----------- | ------------------------------- |
| RUNNING     | 对应的高3位值是111 |
| SHUTDOWN    | 对应的高3位值是000 |
| STOP        | 对应的高3位值是001 |
| TIDYING     | 对应的高3位值是010 |
| TERMINATED  | 对应的高3位值是011 |



线程池各个状态之间的切换如下图所示：

![img](/media/pic/java/threads/juc-executor04-01.jpg)

# 1. RUNNING

(01) 状态说明：线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理。  
(02) 状态切换：线程池的初始化状态是RUNNING。换句话说，线程池被一旦被创建，就处于RUNNING状态！
道理很简单，在ctl的初始化代码中(如下)，就将它初始化为RUNNING状态，并且"任务数量"初始化为0。

    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

 

# 2. SHUTDOWN

(01) 状态说明：线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务。  
(02) 状态切换：调用线程池的shutdown()接口时，线程池由RUNNING -> SHUTDOWN。

 

# 3. STOP

(01) 状态说明：线程池处在STOP状态时，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。  
(02) 状态切换：调用线程池的shutdownNow()接口时，线程池由(RUNNING or SHUTDOWN ) -> STOP。

 

# 4. TIDYING

(01) 状态说明：当所有的任务已终止，ctl记录的"任务数量"为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。  
(02) 状态切换：当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。

当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING。

 

# 5. TERMINATED

(01) 状态说明：线程池彻底终止，就变成TERMINATED状态。  
(02) 状态切换：线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED。

