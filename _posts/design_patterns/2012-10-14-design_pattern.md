---
layout: post
title: "设计模式04之 单例模式(创建模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-14 09:01
---
 
> 本章介绍"单例模式"。

> **目录**  
[1. 单例模式简介](#anchor1)  
[2. 单例模式代码模型](#anchor2)  
[3. 单例模式示例](#anchor3)  

 
<a name="anchor1"></a>
# 1. 单例模式简介

单例模式(Simple Factory)，确保类只有一个实例，而且类自己实例化该实例并向客户端提供该实例。它属于"创建模式"(创建对象的模式)。

单例模式具有以下特点：(01), 类只能有一个实例。(02), 类自行创建实例。(03), 向整个系统提供这个实例。

它的结构图如下所示：

![img](/media/pic/design_patterns/pattern04_01.jpg)
 


<a name="anchor2"></a>
# 2. 单例模式代码模型

单例模式包括：**饿汉式单例模式** 和 **懒汉式单例模式**

**它们的相同点**：它们的单例类中，构造函数都是私有的！这样，就避免了外界利用构造函数创建对象，而只能通过指定的函数来获取对象。此外，由于构造函数是私有的，此类不能被继承！

**它们的不同点**：饿汉式单例模式，是在类被加载时，就创建了类的对象。而在懒汉式单例模式中，在类被第一次引用时，才将自己实例化。

 

## 饿汉式单例模式的代码模型

    public class EagerSingleton {
        private static final EagerSingleton mInstance = new EagerSingleton();

        private EagerSingleton() {}

        public static EagerSingleton getInstance() {
            return mInstance;
        }
    }

在饿汉式单例模式中，在类被加载时，静态变量mInstance就会被初始化。这时候，单例类的唯一实例就被创建出来了。


## 懒汉式单例模式的代码模型

    public class LazySingleton {
        private static LazySingleton mInstance = null;

        private LazySingleton() {}

        synchronized public static LazySingleton getInstance() {
            if (mInstance==null) {
                mInstance = new LazySingleton();
            }
            return mInstance;
        }
    }

在懒汉式单例模式中，在类被加载时，静态变量mInstance并不会被初始化；而只有当getInstance()第1次被调用的时候，mInstance才被初始化。

 
<a name="anchor3"></a>
# 3. 单例模式示例

示例代码：

    class EagerSingleton {
        private static final EagerSingleton mInstance = new EagerSingleton();

        private EagerSingleton() {}

        public static EagerSingleton getInstance() {
            return mInstance;
        }

        public void doSomething() {
            System.out.println(Thread.currentThread().getName()+" do something");
        }
    }

    public class SingletonTest {

        public static void main(String[] args) {
            // 启动5个线程。它们会分别获取EagerSingleton实例，并调用它的doSomething()方法。
            for (int i=0; i<5; i++) {
                new Thread( new Runnable() {
                    @Override
                    public void run() {
                        EagerSingleton es = EagerSingleton.getInstance();
                        es.doSomething();
                    }
                }).start();
            }
        }
    }

运行结果：

    Thread-3 do something
    Thread-4 do something
    Thread-2 do something
    Thread-1 do something
    Thread-0 do something

