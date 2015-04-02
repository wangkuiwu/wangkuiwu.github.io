---
layout: post
title: "设计模式05之 多例模式(创建模式)"
description: "java desigin pattern"
category: pattern
tags: [java, pattern]
date: 2012-10-15 09:01
---
 
> 本章介绍"多例模式"。

> **目录**  
[1. 多例模式简介](#anchor1)  
[2. 多例模式代码模型](#anchor2)  
[3. 多例模式示例](#anchor3)  


 
<a name="anchor1"></a>
# 1. 多例模式简介

多例模式(Multiton)，顾名思义，是指存在一个类由多个相同实例，而且该实例都是该类本身。这个类叫做多例类。

多例模式的特点是： (01), 多例类可以由多个实例。 (02), 多例类必须自己创建、管理自己的实例，并向外界提供自己的实例。

多例模式的结构图如下：

![img](/media/pic/design_patterns/pattern05_01.jpg)
 


<a name="anchor2"></a>
# 2. 多例模式代码模型

    public class Multiton {
        public static final Multiton INSTANCE_01 = new Multiton("instance_01");
        public static final Multiton INSTANCE_02 = new Multiton("instance_02");

        private Multiton(String name) {}
    }

 


<a name="anchor3"></a>
# 3. 多例模式示例

示例代码：

    // 多例类。
    // 包括CHINA, ENGLAND 和 FRANCE 这3个实例。
    class Country {
        public static final Country CHINA   = new Country("chinae");
        public static final Country ENGLAND = new Country("england");
        public static final Country FRANCE  = new Country("france");

        private String name;
        private Country(String name) {
            this.name = name;
        }

        public String getName() {
            return name;
        }
    }

    public class MultitonTest {

        public static void main(String[] args) {
            Country c1 = Country.CHINA;
            Country c2 = Country.ENGLAND;
            Country c3 = Country.FRANCE;

            System.out.println("c1 is: "+c1.getName());
            System.out.println("c2 is: "+c2.getName());
            System.out.println("c3 is: "+c3.getName());
        }
    }

运行结果：

    c1 is: chinae
    c2 is: england
    c3 is: france

