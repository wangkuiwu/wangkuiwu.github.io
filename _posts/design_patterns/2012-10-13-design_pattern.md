---
layout: post
title: "设计模式03之 抽象工厂模式(创建模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-13 09:01
---
 
> 本章介绍"抽象工厂模式"。

> **目录**  
[1. 抽象工厂模式简介](#anchor1)  
[2. 抽象工厂模式代码模型](#anchor2)  
[3. 抽象工厂模式示例](#anchor3)  

 
<a name="anchor1"></a>
# 1. 抽象工厂模式简介

抽象工厂模式(Abstract Factory)，它是所有形态的工厂模式中最为抽象，也是最具有一般性的形态。它属于"创建模式"(创建对象的模式)。它的结构图如下所示：

![img](/media/pic/design_patterns/pattern03_01.jpg)

抽象工厂模式的结构共包括4个组成部分：**抽象工厂(Factory)，具体工厂(ConcreteFactory), 抽象产品(Product)，具体产品(ConcreteProduct)**。

|     组成部分   |       说明      |
| -------------- | --------------- |
| Factory | 它提供了"创建产品"的函数接口，该函数接口由具体工厂ConcreteFactory来实现。Factory可以是接口或者抽象类。 |
| ConcreteFactory | 它实现了Factory的函数接口。 |
| Product | 它是抽象产品，是(许多)不同产品抽象出来的。在抽象工厂模式中，抽象产品不止一个。 |
| ConcreteProduct | 它是具体产品。ConcreteFactory中"创建产品"的函数实现中，实际上是返回的ConcreteProduct实例。 |

**抽象工厂模式与工厂方法模式的区别**: 在工厂方法模式中，"抽象产品"只有一个，而在抽象工厂模式中，"抽象产品"有很多个！在工厂方法模式中，是由"具体工厂"决定返回哪一类产品；然后，抽象工厂中，是由"客户端"决定返回哪一类产品。

 
<a name="anchor2"></a>
# 2. 抽象工厂模式代码模型

代码

    public interface Factory {
        public ProductA newProductA();
        public ProductB newProductB();
    }
    public class ConcreteFactory1 implements Factory {
        public ProductA newProductA() {
            return new ConcreteProductA1();
        }
        public ProductB newProductB();
            return new ConcreteProductB1();
        }
    }
    public class ConcreteFactory2 implements Factory {
        public ProductA newProductA2() {
            return new ConcreteProduct1();
        }
        public ProductB newProductB2();
            return new ConcreteProduct1();
        }
    }
    public interface ProductA {
    }
    public class ProductA1 implements ProductA {
    }
    public class ProductA2 implements ProductA {
    }
    public interface ProductB {
    }
    public class ProductB1 implements ProductB {
    }
    public class ProductB2 implements ProductB {
    }

模型的类图

![img](/media/pic/design_patterns/pattern03_02.jpg)
 

<a name="anchor3"></a>
# 3. 抽象工厂模式示例

假设，我们要实现一个工厂管理系统，记录三星和苹果这两家工厂(Factory)生产的手机(Phone)和电脑(Computer)信息。

已知，三星和苹果都由自己的工厂，分别是"三星工厂(SumFactory)"和"苹果工厂(AppleFactory)"。三星工厂生产"三星手机(SumPhone)"和"三星电脑(SumComputer)"，苹果工厂生产"苹果手机(ApplePhone)"和"苹果电脑(AppleComputer)"。

我们用抽象工厂模式实现该系统，它的设计图如下：

![img](/media/pic/design_patterns/pattern03_03.jpg)

在该系统中，"抽象工厂"，"具体工厂", 抽象产品"和"具体产品"这4个角色分别如下：

## 3.1 抽象工厂

"抽象工厂"是Factory，Factory中定义了"生产手机"以及"生产电脑"的方法。

Factory的源码

    public interface Factory {
        // 生产手机
        public Phone createPhone() ;
        // 生产电脑
        public Computer createComputer() ;
    }

 

## 3.2 具体工厂

"具体工厂"是SumFactory和AppleFactory。SumFactory能返回的"三星手机(SumPhone)"和“三星电脑SumComputer”对象。AppleFactory能返回的"苹果手机(ApplePhone)"和“苹果电脑AppleComputer”对象。

SumFactory的源码

    public class SumFactory implements Factory{
        // 生产三星手机
        public Phone createPhone() {
            return new SumPhone();
        }
        // 生产三星电脑
        public Computer createComputer() {
            return new SumComputer();
        }
    }

AppleFactory的源码

    public class AppleFactory implements Factory{
        // 生产苹果手机
        public Phone createPhone() {
            return new ApplePhone();
        }
        // 生产苹果电脑
        public Computer createComputer() {
            return new AppleComputer();
        }
    }

 

## 2.3 抽象产品

"抽象产品"是"手机(Phone)"和"电脑(Computer)"。手机包括"activate()方法"，而电脑则包括"getOSName()方法"。

Phone的源码

    abstract public class Phone {
        // 激活手机
        abstract public void activate();
    }

Computer的源码

    abstract public class Computer {
        // 获取操作系统名词
        abstract public String getOSName();
    }

 

## 2.4 具体产品

"具体产品"是"三星手机(SumPhone)"，"三星电脑(SumComputer)"；以及 "苹果手机(ApplePhone)"和"苹果电脑(AppleComputer)"。

SumPhone源码

    public class SumPhone extends Phone{
        // 激活手机
        public void activate() {
            System.out.println("activate SumPhone");
        }
    }

SumComputer源码

    public class SumComputer extends Computer{
        // 获取操作系统名词
        public String getOSName() {
            return "Windows";
        }
    }

ApplePhone源码

    public class ApplePhone extends Phone{
        // 激活手机
        public void activate() {
            System.out.println("activate ApplePhone");
        }
    }

AppleComputer源码

    public class AppleComputer extends Computer{
        // 获取操作系统名词
        public String getOSName() {
            return "Mac";
        }
    }

 

## 2.5 客户端测试程序

客户端是"Client"。

Client的源码

    public class Client {

        public static void main(String[] args) {
            // 创建"Factory"对象，该对象是"AppleFactory"的实例
            Factory appleFac = new AppleFactory();
            // 根据工厂实例，创建对应的手机和电脑
            Phone applePhone = appleFac.createPhone();
            Computer appleComputer = appleFac.createComputer();
            // 激活"手机"
            applePhone.activate();
            // 获取"电脑操作系统名词"
            String appleOS =appleComputer.getOSName();
            System.out.println("appleOS is: "+appleOS);

            // 创建"Factory"对象，该对象是"SumFactory"的实例
            Factory sumFac = new SumFactory();
            // 根据工厂实例，创建对应的手机和电脑
            Phone sumPhone = sumFac.createPhone();
            Computer sumComputer = sumFac.createComputer();
            // 激活"手机"
            sumPhone.activate();
            // 获取"电脑操作系统名词"
            String sumOS = sumComputer.getOSName();
            System.out.println("sumOS is: "+sumOS);
        }
    }

运行结果：

    activate ApplePhone
    appleOS is: Mac
    activate SumPhone
    sumOS is: Windows

 
