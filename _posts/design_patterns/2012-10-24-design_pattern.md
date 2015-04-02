---
layout: post
title: "设计模式14之 桥梁(Bridge)模式(结构模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-24 09:01
---
 
> 本章介绍"桥梁模式"。

> **目录**  
[1. 桥梁模式简介](#anchor1)  
[2. 桥梁模式示例](#anchor2)  

 
<a name="anchor1"></a>
# 1. 桥梁模式简介

　　桥梁模式(Bridge)，又称为柄体(Handle and Body)模式或接口(Interface)模式。它属于对象的结构模式，其用意是“将抽象化(Abstraction)与实现化(Implementation)脱耦，使得二者可以独立地变化”。

　　这句话有三个关键词：**"抽象化"、"实现化"和"脱耦"**。

　　• **抽象化**

　　存在于多个实体中的共同的概念联系，就是抽象话。例如，苹果、葡萄、草莓，它们的共同特征就是水果，因此，水果是它们的抽象化存在；圆形、正方形、三角形，它们的共同特征就是形状，因此，形状就是它们的抽象话。

　　通常情况下，一组对象如果具有相同的特征，那么它们就可以通过一个共同的类来描述。如果一些类具有相同的特征，往往可以通过一个共同的抽象类来描述。

　　• **实现化**

　　抽象化给出的具体实现，就是实现化。

　　一个类的实例就是这个类的实例化，一个具体子类是它的抽象超类的实例化。

　　• **脱耦**

　　所谓耦合，就是两个实体的行为的某种强关联。而将它们的强关联去掉，就是耦合的解脱，或称脱耦。在这里，脱耦是指将抽象化和实现化之间的耦合解脱开，或者说是将它们之间的强关联改换成弱关联。

　　所谓强关联，就是在编译时期已经确定的，无法在运行时期动态改变的关联；所谓弱关联，就是可以动态地确定并且可以在运行时期动态地改变的关联。显然，在Java语言中，继承关系是强关联，而聚合关系是弱关联。

　　将两个角色之间的继承关系改为聚合关系，就是将它们之间的强关联改换成为弱关联。因此，桥梁模式中的所谓脱耦，就是指在一个软件系统的抽象化和实现化之间使用聚合关系而不是继承关系，从而使两者可以相对独立地变化。这就是桥梁模式的用意。


<br/>
桥梁模式的UML类图

![img](/media/pic/design_patterns/pattern14_01.jpg)

从中可以看出，桥接模式含有两个等级结构：  
(01) 由抽象化角色和修正抽象化角色组成的抽象化等级结构。  
(02) 由实现化角色和两个具体实现化角色所组成的实现化等级结构。

桥梁模式包含四种角色： **抽象化(Abstraction)，修正抽象化(RefinedAbstraction)，实现化(Implementor)，具体实现化(ConcreteImplementor)**。  
• 抽象化: 抽象化给出的定义，并保存一个对实现化对象的引用。  
• 修正抽象化: 扩展抽象化角色，改变和修正父类对抽象化的定义。  
• 实现化: 这个角色给出实现化角色的接口，但不给出具体的实现。必须指出的是，这个接口不一定和抽象化角色的接口定义相同，实际上，这两个接口可以非常不一样。实现化角色应当只给出底层操作，而抽象化角色应当只给出基于底层操作的更高一层的操作。  
• 具体实现化: 这个角色给出实现化角色接口的具体实现。

　　对象是对行为的封装，而行为是由方法实现的。在这个示意性系统里，抽象化等级结构中的类封装了operation()方法；而实现化等级结构中的类封装的是operationImpl()方法。当然，在实际的系统中往往会有多于一个的方法。

　　抽象化等级结构中的方法通过向对应的实现化对象的委派实现自己的功能，这意味着抽象化角色可以通过向不同的实现化对象委派，来达到动态地转换自己的功能的目的。




## 示意代码

抽象化

    public abstract class Abstraction {
        
        protected Implementor impl;
        
        public Abstraction(Implementor impl){
            this.impl = impl;
        }
        // 商业方法
        public void operation(){
            
            impl.operationImpl();
        }
    }

修正抽象化角色

    public class RefinedAbstraction extends Abstraction {
        
        public RefinedAbstraction(Implementor impl) {
            super(impl);
        }
    }

实现化角色

    public abstract class Implementor {
        // 方法的实现化声明
        public abstract void operationImpl();
    }

具体实现化角色

    public class ConcreteImplementorA extends Implementor {

        // 方法的实现化实现
        @Override
        public void operationImpl() {
            // something you want to do
        }

    }
    public class ConcreteImplementorB extends Implementor {

        // 方法的实现化实现
        @Override
        public void operationImpl() {
            // something you want to do
        }
    }

 
<a name="anchor2"></a>
# 2. 桥梁模式示例

　　**问题**

　　空中巴士(Airbus)、波音(Boeing)和麦道(MD)都是飞机制造商，它们都生成载客飞机(PassengerPlane)和载货飞机(CargoPlane)。现在需要设计一个系统，描述这些飞机制造商以及它们所制造的飞机种类。

 

　　**设计方案一(不使用桥梁模式)**

　　系统是关于飞机的，因此可以设计一个总的飞机接口，叫做Airplane。其它所有的飞机都是这个总接口的字接口或者具体实现。

　　下面是这个方案的设计图，可以看出，这是一个不太高明的设计，导致了理不清的关系。

![img](/media/pic/design_patterns/pattern14_02.jpg)

　　在这个设计方案里面，出现了两个子接口，分别代表客户和货机。所有的具体飞机又要继承自Airbus，Boeing和MD等超类。这样一类，每个具体飞机都带有两个超类：飞机制造商类型，客、货机类型。

 

　　**设计方案二(使用桥梁模式)**

　　使用桥梁模式的关键在于准确的找出这个系统的抽象化角色和具体化角色。从系统所面对的问题不难看出，代表飞机的抽象化是它的类型，也就是"客户"或者"货机"；而代表飞机的实现化的则是飞机的制造商。

使用桥梁模式，对应的UML类图如下：

![img](/media/pic/design_patterns/pattern14_03.jpg)

## 2.1 抽象化类

    abstract public class Airplane {
        // 飞机制造商
        protected AirplaneMaker maker;

        // 指定飞机制造商
        public Airplane(AirplaneMaker maker) {
            this.maker = maker;
        }

        public void fly() {
            // 调用AirplaneMaker的produce()方法
            maker.produce();
        }
    }

## 2.2 修正抽象化

PassengerPlane代码

    public class PassengerPlane extends Airplane {
        public PassengerPlane(AirplaneMaker maker) {
            super(maker);
        }

        public void fly() {
            super.fly();
            System.out.println("PassengerPlane fly.");
        }
    }

CargoPlane代码

    public class CargoPlane extends Airplane {

        public CargoPlane(AirplaneMaker maker) {
            super(maker);
        }

        public void fly() {
            super.fly();
            System.out.println("CargoPlane fly.");
        }
    }

## 2.3 实现化

AirplaneMaker代码

    abstract public class AirplaneMaker {
        // 生产飞机
        public abstract void produce() ;
    }

## 2.4 具体实现化

Airbus代码

    public class Airbus extends AirplaneMaker {
        public void produce() {
            System.out.println("Airbus produced.");
        }
    }

Boeing代码

    public class Boeing extends AirplaneMaker {
        public void produce() {
            System.out.println("Boeing produced.");
        }
    }

MD代码

    public class MD extends AirplaneMaker {
        public void produce() {
            System.out.println("MD produced.");
        }
    }

## 2.5 客户端测试程序

    public class Client {

        public static void main(String[] args) {
            // "飞机制造商"为Airbus
            AirplaneMaker maker = new Airbus();
            // "飞机类型"为PassengerPlane
            Airplane plane = new PassengerPlane(maker);
            // 飞机飞行
            plane.fly();

            // "飞机制造商"为MD
            maker = new MD();
            // "飞机类型"为CargoPlane
            plane = new CargoPlane(maker);
            // 飞机飞行
            plane.fly();
        }
    }

 

运行结果：

    Airbus produced.
    PassengerPlane fly.
    MD produced.
    CargoPlane fly.

 
