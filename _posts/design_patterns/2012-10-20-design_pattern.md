---
layout: post
title: "设计模式10之 装饰(Decorator)模式(结构模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-20 09:01
---
 
> 本章介绍"装饰模式"。

> **目录**  
[1. 装饰模式简介](#anchor1)  
[2. 装饰模式示例](#anchor2)  

 
<a name="anchor1"></a>
# 1. 装饰模式简介

装饰(Decorator)模式又名包装(Wrapper)模式。装饰模式以对客户端透明的方式扩展对象的功能，是继承关系的一个替代方案。

它以对客户透明的方式动态地给一个对象附加上更多的责任。换言之，客户端并不会觉得对象在装饰前和装饰后有什么不同。装饰模式可以在不使用创造更多子类的情况下，将对象的功能加以扩展。

装饰模式的类图如下：

![img](/media/pic/design_patterns/pattern10_01.jpg)

装饰模式包含了三个角色: **抽象构件(Component)，具体构件(ConcreteComponent) ，装饰(Decorator) 和 具体装饰(ConcreteDecorator)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象构件(Component) | 给出一个抽象接口，以规范准备接收附加责任的对象。 |
| 具体构件(ConcreteComponent) | 定义一个将要接收附加责任的类。 |
| 装饰(Decorator) | 持有一个构件(Component)对象的实例，并定义一个与抽象构件接口一致的接口。 |
| 具体装饰(ConcreteDecorator) | 负责给构件对象“贴上”附加的责任。 |

 

示意代码

    public interface Component {
        // 商业方法
        public void sampleOperation();
    }

    public class ConcreteComponent implements Component {
        // 构造函数
        public ConcreteComponent() {
            // write your code here
        }

        @Override
        public void sampleOperation() {
            // 写相关的业务代码
        }
    }

    public class Decorator implements Component{
        private Component component;
         
        // 构造函数
        public Decorator(Component component){
            // write your code here
        }

        // 构造函数
        public Decorator(Component component){
            this.component = component;
        }

        @Override
        public void sampleOperation() {
            // 商业方法，委派给构件
            component.sampleOperation();
        }
    }

    public class ConcreteDecorator extends Decorator {

        @Override
        public void sampleOperation() {
            super.sampleOperation();
            // write your code here
        }
    }

需要指出的是：  
(01) 在装饰类Decorator中，有一个私有树形component，其数据结构是构件(Component)。  
(02) 装饰类Decorator实现了构件(Component)接口。  
(03) Decorator类中，接口的每一个实现方法都委派给父类，但并不单纯地委派，而是有功能的增强。

 
<a name="anchor2"></a>
# 2. 装饰模式示例

下面，通过"齐天大圣"的例子来对装饰模式进行说明。

孙悟空有七十二般变化，他的每一种变化都给他带来一种附加的本领。他变成鱼儿时，就可以到水里游泳；他变成鸟儿时，就可以在天上飞行。

在装饰模式中，Component的角色便由鼎鼎大名的齐天大圣扮演；ConcreteComponent的角色属于大圣的本尊，就是猢狲本人；Decorator的角色由大圣的七十二变扮演。而ConcreteDecorator的角色便是花、鸟、鱼、虫等七十二般变化。它的类图如下：

![img](/media/pic/design_patterns/pattern10_02.jpg)
 

示例代码

## 2.1 抽象构件(Component)

抽象构件是"齐天大圣"，它包行了move()接口。

    abstract public class GreatSage {
        public abstract void move();
    }

 

## 2.2 具体构件(ConcreteComponent)
具体构件属于大圣的本尊，就是孙悟空。

    public class Monkey extends GreatSage {
        @Override
        public void move() {
            System.out.println("Monkey Move");
        }
    }

 

## 2.3 装饰(Decorator)

装饰由大圣的七十二变扮演。

    public class Change extends GreatSage {
        private GreatSage sage;
        
        public Change(GreatSage sage){
            this.sage = sage;
        }

        @Override
        public void move() {
            sage.move();
        }
    }

 

## 2.4 具体装饰(ConcreteDecorator)。

花

    public class Flower extends Change {
        
        public Flower(GreatSage sage) {
            super(sage);
        }

        @Override
        public void move() {
            System.out.println("Flower Move");
        }
    }

鸟

    public class Bird extends Change {
        
        public Bird(GreatSage sage) {
            super(sage);
        }

        @Override
        public void move() {
            System.out.println("Bird Move");
        }
    }

 

鱼

    public class Fish extends Change {
        
        public Fish(GreatSage sage) {
            super(sage);
        }

        @Override
        public void move() {
            System.out.println("Fish Move");
        }
    }

 

虫

    public class Insect extends Change {
        
        public Insect(GreatSage sage) {
            super(sage);
        }

        @Override
        public void move() {
            System.out.println("Insect Move");
        }
    }

 

## 2.5 客户端测试程序

    public class Client {

        public static void main(String[] args) {
            GreatSage sage = new Monkey();

            // 1. "齐天大圣"
            sage.move();

            // 2. "齐天大圣"变成"鱼"之后，再变成"鸟"
            GreatSage fish = new Fish(sage);
            GreatSage bird = new Bird(fish);
            bird.move(); 

            // 3. "齐天大圣"变成"昆虫"之后，再变成"花"
            GreatSage flower = new Flower(new Insect(sage));
            flower.move(); 
        }
    }

运行结果：

    Monkey Move
    Fish Move
    Flower Move

结果说明：上面的例子的2中，系统把大圣从一只猢狲装饰成了一只鸟儿（把鸟儿的功能加到了猢狲身上），然后又把鸟儿装饰成了一条鱼儿（把鱼儿的功能加到了猢狲+鸟儿身上，得到了猢狲+鸟儿+鱼儿）。如下图所示：

![img](/media/pic/design_patterns/pattern10_03.jpg)


![img](/media/pic/design_patterns/pattern10_04.jpg)
　　
 

 
