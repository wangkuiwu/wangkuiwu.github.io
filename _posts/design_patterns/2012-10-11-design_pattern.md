---
layout: post
title: "设计模式01之 简单工厂模式(创建模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-11 09:01
---

 
> 本章介绍"简单工厂模式"。

> (01) 本文是在《Java与模式》的学习总结文章！  
(02) 文章中的UML的相关内容(包括类图说明和绘图工具等)，可以参考"UML系列" 文章。

> **目录**  
[1. 简单工厂模式简介](#anchor1)  
[2. 简单工厂模式代码模型](#anchor2)  
[3. 简单工厂模式示例](#anchor3)  


<a name="anchor1"></a>
# 1. 简单工厂模式简介

简单工厂模式(Simple Factory)，又被称为"静态工厂方法模式"。它属于"创建模式"(创建对象的模式)，并且是"工厂方法"模式的一种特殊实现。

通常，我们利用简单工厂模式来进行类的创建。例如，获取线程池对象，就是通过简单工厂模式来实现的。它的结构图如下所示：

![img](/media/pic/design_patterns/pattern01_01.jpg)

简单工厂模式的结构共包括3个组成部分：**工厂(Factory)，抽象产品(Product)，具体产品(ConcreteProduct)**。

 
|     组成部分   |       说明      |
| -------------- | --------------- |
|  工厂          | 工厂是简单工厂模式的核心，提供了对外接口。客户端或其它程序要获取Product对象，都是通过Factory的接口来获取的。 |
|  抽象产品      | 抽象产品是(许多)不同产品抽象出来的。Product可以是接口或者抽象类。 |
|  具体产品      | 工厂中返回的产品对象，实际上是通过ConcreteProduct来创建的。 |

 

 
<a name="anchor2"></a>
# 2. 简单工厂模式代码模型

    public class Factory {
        public static Product newInstance() {
            return new ConcreteProduct();
        }
    }
    public abstract Product {
    }
    public class ConcreteProduct extends Product {
        public ConcreteProduct() {}
    }

模型的类图

![img](/media/pic/design_patterns/pattern01_02.jpg)
 

<a name="anchor3"></a>
# 3. 简单工厂模式示例

假设现在有一个水果工厂，能够生成各种各样的水果。目前能生产的水果包括苹果，葡萄和草莓。

我们通过"简单工厂模式"描述该问题，它的UML类图如下：

![img](/media/pic/design_patterns/pattern01_03.jpg)

## 3.1 工厂类

工厂类是"FruitFactory"。FruitFactory是水果工厂，水果工厂会生成水果。

FruitFactory的源码

    public class FruitFactory {

        public static Fruit newInstance(String name) 
            throws BadFruitException {
            // 如果name等于"apple"(忽略大小写)，则返回苹果。
            if ("apple".equalsIgnoreCase(name)) {
                return new Apple();
            // 如果name等于"grape"(忽略大小写)，则返回葡萄。
            } else if ("grape".equalsIgnoreCase(name)) {
                return new Grape();
            // 如果name等于"strawberry"(忽略大小写)，则返回草莓。
            } else if ("strawberry".equalsIgnoreCase(name)) {
                return new Strawberry();
            // 其它情况，则抛出异常。
            } else {
                throw new BadFruitException("Bad fruit request!");
            }
        }
    }

 

## 2.2 抽象产品类

Product类是"Fruit"。Fruit代表水果，它是抽象类，包含水果的基本特征：生长，种植，收获。

Fruit的源码

    public abstract class Fruit {
        abstract void grow();    // 生长
        abstract void harvest(); // 收获
        abstract void plant();   // 种植
    }

 

## 2.3 具体产品类

具体产品类是"Apple", "Grape"和"Strawberry"。它们是3种具体的水果，分别代表苹果，葡萄和草莓。

Apple的源码

    // Apple实现Fruit的函数接口，并且Apple中有私有成员age和私有方法log。
    public class Apple extends Fruit {
        private int age;

        public void grow() {
            log("Apple grow()");
        }
        public void harvest() {
            log("Apple harvest()");
        }
        public void plant() {
            log("Apple plant()");
        }
        public void setAge(int age) {
            this.age = age;
        }
        public int getAge() {
            return age;
        }
        private void log(String msg) {
            System.out.println(msg);
        }
    }

Grape的源码

    // Grape仅仅只实现Fruit的函数接口。
    public class Grape extends Fruit {
        public void grow() {
            System.out.println("Grape grow()");
        }
        public void harvest() {
            System.out.println("Grape harvest()");
        }
        public void plant() {
            System.out.println("Grape plant()");
        }
    }

Strawberry的源码

    // Strawberry实现Fruit的函数接口，并且Strawberry中有私有方法log
    public class Strawberry extends Fruit {

        public void grow() {
            log("Strawberry grow()");
        }
        public void harvest() {
            log("Strawberry harvest()");
        }
        public void plant() {
            log("Strawberry plant()");
        }
        private void log(String msg) {
            System.out.println(msg);
        }
    }

 

## 2.4 客户端测试程序

客户端是"Client"。

Client的源码

    public class Client {

        public static void main(String[] args) {
            try {
                Fruit apple = FruitFactory.newInstance("Apple");
                apple.plant();
                apple.grow();
                apple.harvest();

                Fruit grape = FruitFactory.newInstance("Grape");
                grape.plant();
                grape.grow();
                grape.harvest();

                Fruit strawberry = FruitFactory.newInstance("strawberry");
                strawberry.plant();
                strawberry.grow();
                strawberry.harvest();

                Fruit error = FruitFactory.newInstance("error");
                error.plant();
                error.grow();
                error.harvest();
            } catch (BadFruitException e) {
                e.printStackTrace();
            }
        }
    }

运行结果：

    Apple plant()
    Apple grow()
    Apple harvest()
    Grape plant()
    Grape grow()
    Grape harvest()
    Strawberry plant()
    Strawberry grow()
    Strawberry harvest()
    BadFruitException: Bad fruit request!
        at FruitFactory.newInstance(FruitFactory.java:17)
        at Client.main(Client.java:21)

结果说明：

Client成功创建了Apple, Grape, Strawberry这3个水果对象，并调用了它们的方法；然后创建error时，由于不存在error对应的水果，因此抛出异常。

(01) Fruit，是抽象产品。Fruit中声明了grow(), harvest(), plant()这3个函数接口，它们是水果共用拥有的行为。

(02) Apple, Grape和Strawberry这三个类是具体产品。  
Apple -- 实现Fruit的函数接口，并且Apple中有私有成员age和私有方法log。  
Grape -- 仅仅只实现Fruit的函数接口。  
Strawberry -- 实现Fruit的函数接口，并且Strawberry中有私有方法log

(03) FruitFactory，是工厂类。通过它的newInstance()方法，我们可以获取相应的Fruit对象。若要获取的水果不存在，则抛出BadFruitException异常。BadFruitException是Exception的子类。

