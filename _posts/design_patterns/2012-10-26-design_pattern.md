---
layout: post
title: "设计模式16之 策略(Strategy)模式(行为模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-26 09:01
---
 

> 本章介绍"策略模式"。

> **目录**  
[1. 策略模式简介](#anchor1)  
[2. 策略模式示例](#anchor2)  

 
<a name="anchor1"></a>
# 1. 策略模式简介

策略(Strategy)模式属于对象的行为模式。其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换。策略模式使得算法可以在不影响到客户端的情况下发生变化。


UML类图

![img](/media/pic/design_patterns/pattern16_01.jpg)

这个模式涉及到三个角色：**环境(Context)，抽象策略(Strategy)，具体策略(ConcreteStrategy)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 环境 | 持有一个Strategy的引用。 |
| 抽象策略 | 这是一个抽象角色，通常由一个接口或抽象类实现。此角色给出所有的具体策略类所需的接口。 |
| 具体策略 | 包装了相关的算法或行为。 |

示意代码

    public class Context {
        // 策略对象
        private Strategy strategy;

        // 构造函数
        public Context(Strategy strategy){
            this.strategy = strategy;
        }

        // 策略方法
        public void contextInterface(){
            strategy.strategyInterface();
        }
    }

    abstract public class Strategy {
        // 策略方法
        public abstract void strategyInterface();
    }

    public class ConcreteStrategy extends Strategy {

        @Override
        public void strategyInterface() {
            // write your algorithm code here
        }
    }

 


<a name="anchor2"></a>
# 2. 策略模式示例

假设现在要设计一个贩卖各类书籍的电子商务的购物车系统。下面是图书的三种折扣算法。

算法一：对图书没有折扣。  
算法二：对图书提供一个固定值为5元的折扣。  
算法三：对图书提供一个八折的折扣。

使用策略模式描述的话，这些不同的算法都是不同的具体策略角色：用一个NoDiscountStrategy对象描述"算法一"；用一个FlatRateStrategy对象描述"算法二"；用一个PercentageStrage对象描述"算法三"。

## 2.1 抽象策略

    abstract public class DiscountStrategy {

        // 价格
        protected int price = 0;
        // 策略
        protected int copies = 0;

        public abstract int calculateDiscount();

        public DiscountStrategy(int price, int copies) {
            this.price = price;
            this.copies = copies;
        }
    }

## 2.2 具体策略

算法一(无折扣)

    public class NoDiscountStrategy extends DiscountStrategy {

        public NoDiscountStrategy(int price, int copies) {
            super(price, copies);
        }

        public int calculateDiscount() {
            return copies*price;
        }
    }

算法二(优惠5元)

    public class FlatRateStrategy extends DiscountStrategy {

        private int discard = 5;

        public FlatRateStrategy(int price, int copies) {
            super(price, copies);
        }

        public void setDiscard() {
            this.discard = discard;
        }

        public int getDiscard() {
            return discard;
        }

        public int calculateDiscount() {
            return copies*(price - discard);
        }
    }

算法三(打8折)

    public class PercentageStrategy extends DiscountStrategy {

        // 折扣比例
        private float percent = 0.8f;

        public PercentageStrategy(int price, int copies) {
            super(price, copies);
        }

        public void setPercent() {
            this.percent = percent;
        }

        public float getPercent() {
            return percent;
        }

        public int calculateDiscount() {
            return (int)(copies*price*percent);
        }
    }

## 2.3 客户端

    public class Client {

        public static void main(String[] args) {
            // 第一种策略
            DiscountStrategy strategy = new NoDiscountStrategy(60, 5);
            System.out.println("total(60,5)="+strategy.calculateDiscount());

            strategy = new FlatRateStrategy(80, 5);
            System.out.println("total(80,5)="+strategy.calculateDiscount());

            strategy = new PercentageStrategy(100, 3);
            System.out.println("total(100,4)="+strategy.calculateDiscount());

        }
    }

运行结果：

    total(60,5)=300
    total(80,5)=375
    total(100,4)=240

 
