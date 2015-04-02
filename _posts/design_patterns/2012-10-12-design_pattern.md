---
layout: post
title: "设计模式02之 工厂方法模式(创建模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-12 09:01
---
 
> 本章介绍"工厂方法模式"。

> **目录**  
[1. 工厂方法模式简介](#anchor1)  
[2. 工厂方法模式代码模型](#anchor2)  
[3. 工厂方法模式示例](#anchor3)  
[4. 工厂方法模式和简单工厂模式的比较](#anchor4)  


 
<a name="anchor1"></a>
# 1. 工厂方法模式简介

工厂方法模式(Factory Method)，又被称为"虚构造子模式"或"多态性工厂模式"。它属于"创建模式"(创建对象的模式)。

它的用意是提供一个"创建产品对象"的接口，而将这些接口的实现推迟到它的子类中去。因此它具有两个明显的优点：  
(1) 定义创建对象的接口,封装了对象的创建。  
(2) 使得具体化类的工作延迟到了子类中。

下面看看它的结构图：

![img](/media/pic/design_patterns/pattern02_01.jpg)

工厂方法模式的结构共包括4个组成部分：**抽象工厂(Factory)，具体工厂(ConcreteFactory), 抽象产品(Product)，具体产品(ConcreteProduct)。**

|     组成部分   |       说明      |
| -------------- | --------------- |
| Factory | 它提供了"创建产品"的函数接口，该函数接口由具体工厂ConcreteFactory来实现。Factory可以是接口或者抽象类。 |
| ConcreteFactory | 它实现了Factory的函数接口。在上面的结构图中，给出了ConcreteFactory1和ConcreteFactory2两个具体工厂类。 |
| Product | 它是抽象产品。 |
| ConcreteProduct | 它是具体产品，实现了Product中的函数接口。ConcreteFactory中"创建产品"的函数实现中，实际上是返回的ConcreteProduct实例。在上面的结构图中，给出了ConcreteProduct1和ConcreteProduct2两个具体产品类。 |

 

<a name="anchor2"></a>
# 2. 工厂方法模式代码模型

代码

    public interface Factory {
        public Product newInstance() ;
    }
    public class ConcreteFactory1 implements Factory {
        public Product newInstance() {
            return new ConcreteProduct1();
        }
    }
    public class ConcreteFactory2 implements Factory {
        public Product newInstance() {
            return new ConcreteProduct2();
        }
    }
    public interface Product {
    }
    public class ConcreteProduct1 implements Product {
        public ConcreteProduct1() {
            // do something
        }
    }
    public class ConcreteProduct2 implements Product {
        public ConcreteProduct2() {
            // do something
        }
    }

模型的类图

![img](/media/pic/design_patterns/pattern02_02.jpg)

以上模型中，对于"抽象工厂"和"抽象产品"都只给出了两个实现类；在实际中，情况可能比这复杂许多。

 
<a name="anchor3"></a>
# 3. 工厂方法模式示例

将"简单工厂模式"中的示例，改为由"工厂方法模式"实现。它的UML类图如下：

![img](/media/pic/design_patterns/pattern02_03.jpg)

## 3.1 抽象工厂类

抽象工厂类类是"FruitFactory"。FruitFactory是水果工厂，水果工厂会生成苹果，葡萄和草莓这3种水果。

FruitFactory的源码

    abstract public class FruitFactory {
        public abstract Fruit newInstance();
    }

 

## 3.2 具体工厂类

具体工厂类是"AppleFactory", "GrapeFactory"和"StrawberryFactory"；它们都继承于FruitFactory。AppleFactory生产苹果，GrapeFactory生成葡萄，Strawberry生成草莓。

AppleFactory的源码

    public class AppleFactory extends FruitFactory {
        public Fruit newInstance() {
            return new Apple();
        }
    }

GrapeFactory的源码

    public class GrapeFactory extends FruitFactory {
        public Fruit newInstance() {
            return new Grape();
        }
    }

Strawberry的源码

    public class StrawberryFactory extends FruitFactory {
        public Fruit newInstance() {
            return new Strawberry();
        }
    }

 
## 2.3 抽象产品类

抽象产品类是"Fruit"。Fruit代表水果，它是抽象类，包含水果的基本特征：生长，种植，收获。

Fruit的源码

    public abstract class Fruit {
        abstract void grow();    // 生长
        abstract void harvest(); // 收获
        abstract void plant();   // 种植
    }


## 2.4 具体产品类

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

 

## 2.5 客户端测试程序

客户端是"Client"。

Client的源码

    public class Client {

        public static void main(String[] args) {
            // 创建"抽象工厂FruitFactory"对象，该对象是"具体工厂AppleFactory"的实例
            FruitFactory appleFac = new AppleFactory();
            // 根据工厂实例，创建对应的产品
            Fruit apple = appleFac.newInstance();
            apple.plant();
            apple.grow();
            apple.harvest();

            // 创建"抽象工厂FruitFactory"对象，该对象是"具体工厂GrapeFactory"的实例
            FruitFactory grapeFac = new GrapeFactory();
            // 根据工厂实例，创建对应的产品
            Fruit grape = grapeFac.newInstance();
            grape.plant();
            grape.grow();
            grape.harvest();

            // 创建"抽象工厂FruitFactory"对象，该对象是"具体工厂StrawberryFactory"的实例
            FruitFactory strawberryFac = new StrawberryFactory();
            // 根据工厂实例，创建对应的产品
            Fruit strawberry = strawberryFac.newInstance();
            strawberry.plant();
            strawberry.grow();
            strawberry.harvest();
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

 
<a name="anchor4"></a>
# 4. 工厂方法模式和简单工厂模式的比较

在简单共存模式中，"工厂类"处于实例化产品的中心位置；它知道每一个产品，决定了哪一个产品类应该被实例化。如果有新的产品添加到系统中，就需要对应的修改"工厂类"。
而工厂方法模式的出现，既保持了简单工厂模式的优点，又客服了它的缺点。在工厂方法模式中，"工厂类"不再负责产品的实例化，而是将实例化工作交给它的子类(具体工厂)去完成。这样，当有新的产品添加到系统中时，就不需要修改"工厂类"，而只要添加对应的"具体工厂"类即可!

简单工厂模式相当于工厂方法模式的特殊形式。即，当工厂方法模式中，只有一个"具体工厂"存在时，将"抽象工厂"和"具体工厂"合并成一个类；接着，将实例化产品的方法改为静态方法。此时，就得到了简单工厂模式。

反之，工厂方法模式也相当于"多态"的简单工厂模式。

 
