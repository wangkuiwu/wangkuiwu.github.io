---
layout: post
title: "设计模式07之 原型模式(创建模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-17 09:01
---
 
> 本章介绍"原型模式"。

> **目录**  
[1. 原型模式简介](#anchor1)  
[2. 简单形式的原型模式](#anchor2)  
[3. 登记形式的原型模式](#anchor3)  

 
<a name="anchor1"></a>
# 1. 原型模式简介

原型模式是"创建模式"(创建对象的模式)。通过给出一个原型对象来指明所要创建的对象的类型，然后用复制这个原型对象的办法创建出更多同类型的对象。

Java语言的支持支持创建模式。在Java中Object是所有类的父类，而Object中提供了clone()方法。clone()会通过调用本地方法实现对象的复制。它的源码如下：

    protected native Object clone() throws CloneNotSupportedException;

clone()定义了Object中，就相当于Java中的任何类都继承了clone()方法。但是，如果要类支持克隆的话，必须要声明该类实现了Cloneable接口。否则，调用该类的clone()方法时，会抛出CloneNotSupportedException异常。

 

使用场景: 系统的产品是动态加载的，而产品类具有一定的等级结构。 例如，数据库中存储了很多数据，有时我们需要将其中的一些数据取出来存放到新的表格中。或者，有时候我们需要对比一个对象在处理前/后的姿态；在处理之前，可以克隆该对象，然后和处理之后的对象比较。

原型模式有两种表现形式：**简单形式** 和 **登记形式**。  
如果需要创建的原型对象数目少而且比较固定的华，就采用"简单形式"；如果要创建的原型对象数目不确定的华，就采用"登记形式"。



<a name="anchor2"></a>
# 2. 简单形式的原型模式

类图

![img](/media/pic/design_patterns/pattern07_01.jpg)

它涉及到3个角色：**客户(Client), 抽象原型(Prototype) 和 具体原型(ConcretePrototype)**。

|     角色   |       说明      |
| ---------- | --------------- |
| Client | 客户类提出创建对象的请求。 |
| Prototype | 抽象原型给出所有的具体原型所需要实现的函数接口。 |
| ConcreteType | 被复制的对象，实现了Prototype中的函数接口。 |


代码模型

    public interface Prototype extends Cloneable {
        Prototype clone();
    }
    public class ConcretePrototype implements Prototype {
        public Object clone() {
            try {
                return super.clone();
            } catch (CloneNotSupportedException e) {
                return null;
            }
        }
    }
    public class Client {
        private Prototype prototype;
        public void operation(Prototype example) {
            Prototype p = (Prototype) example.clone();
        }
    }

 
<a name="anchor3"></a>
# 3. 登记形式的原型模式

UML类图

![img](/media/pic/design_patterns/pattern07_02.jpg)

它涉及到4个角色：**客户(Client), 抽象原型(Prototype), 具体原型(ConcretePrototype) 和 PrototypeManager(原型管理器)**。

|     角色   |       说明      |
| ---------- | --------------- |
| Client | 客户类提出创建对象的请求。 |
| Prototype | 抽象原型给出所有的具体原型所需要实现的函数接口。 |
| ConcreteType | 被复制的对象，实现了Prototype中的函数接口。 |
| PrototypeManager | 创建具体原型类的对象，并记录每一个被创建的对象。 |

 

代码模型

    public interface Prototype extends Cloneable {
        Prototype clone();
    }
    public class ConcretePrototype implements Prototype {
        public synchronized Object clone() {
            Prototype temp = null;
            try {
                temp = (Prototype)super.clone();
                return temp;
            } catch (CloneNotSupportedException e) {
                return null;
            } finally {
                return temp;
            }
        }
    }
    public class PrototypeManager {
        private Vector objects = new Vector();

        public void add(Prototype object) {
            objects.add(object);
        }
        public Prototype get(int i) {
            return (Prototype) objects.get(i);
        }
        public int getSize() {
            return objects.size();
        }
    }
    public class Client {
        private PrototypeManager mgr;
        private Prototype prototype;
        public void registerPrototype() {
            prototype = new ConcretePrototype();
            Prototype copytype = (Prototype)prototype.clone();
            mgr.add(copytype);
        }
    }

 
