---
layout: post
title: "设计模式11之 代理(Proxy)模式(结构模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-21 09:01
---
 
> 本章介绍"代理模式"。

> **目录**  
[1. 代理模式简介](#anchor1)  
[2. 代理模式示例](#anchor2)  

 
<a name="anchor1"></a>
# 1. 代理模式简介

代理(Proxy)模式是对象的结构模式。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。

一个比较典型的例子就是"Windows的快捷方式"。Windows快捷方式，可以使任何对象同时出现在多个地方而不必修改原对象。对快捷方式的调用完全与对原对象的调用一样，换言之，快捷方式对客户端是完全透明的。


代理模式的类图如下：

![img](/media/pic/design_patterns/pattern11_01.jpg)

代理模式中共包含了三个角色: **抽象主体(Subject)，代理主题(ProxySubject)，真实主题(RealSubject)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象主题角色 | 声明了真实主题和代理主题的共同接口，这样一来在任何可以使用真实主题的地方都可以使用代理主题。 |
| 代理主题角色 | 代理主题内部含有真实主题的引用，从而可以在任何时候操作真实主题；代理主题提供一个与真实主题相同的接口，以便可以在任何时候替代真实主题。代理主题通常在客户端调用传递给真实主题之前或之后，执行某个操作，而不是单纯地将调用传递给真实主题。 |
| 真实主题角色 | 定义了代理主题所代表的真实主题。 |

 

示意代码

    abstract public class Subject {
        // 声明一个抽象的请求方法
        public abstract void request();
    }

    public class RealSubject extends Subject {
        @Override
        public void request() {
            System.out.println("From real subject");
        }
    }

    public class ProxySubject extends Subject{
        RealSubject realSubject = new RealSubject();

        @Override
        public void request() {
            preRequest();
            realSubject.request();        
            postRequest();
        }

        // 请求前的操作
        private void preRequest() {
            // something you want to do before requesting
        }

        // 请求后的操作
        private void postRequest() {
            // something you want to do after requesting
        }
    }

 

若要使用代理主题，则可以通过以下代码：

    Subject subject = new ProxySubject();
    subject.request();

从中可以看出代理模式是怎样工作的。首先，代理模式并不改变主题的接口，因为代理模式的用意是不让客户端感觉到代理的存在；其次，代理使用委派将客户端的调用委派给真实的主题对象；换言之，代理主题起到的是一个传递请求的作用。最后，代理主题在传递请求之前和之后都可以执行特定的操作，而不是单纯的传递请求。

它的时序图如下：

![img](/media/pic/design_patterns/pattern11_02.jpg)



<a name="anchor2"></a>
# 2. 代理模式示例

下面，通过"高老庄悟空降八戒"的例子来对装饰模式进行说明。

悟空假扮成高小姐去见猪八戒。通过代理模式去分析，"高小姐的神貌"是抽象主题，高小姐本人是真实主题，而悟空是代理主题，他巧妙的实现了"高小姐的神貌"；猪八戒就是客户端，猪八戒根本分不清"悟空假扮的高小姐"和"高小姐本人"。

对应的类图如下：

![img](/media/pic/design_patterns/pattern11_03.jpg)

下面是各个角色对应的类。

## 2.1 抽象主体

抽象主题是"高小姐的神貌"，它对应的类是AbstractAppearance。

    abstract public class AbstractAppearance {
        public abstract void smile();
    }


## 2.2 真实主题

真实主题是"高小姐本人"，它对应的类是MissGao。

    public class MissGao extends AbstractAppearance {
        @Override
        public void smile() {
            System.out.println("Miss Gao smiles.");
        }
    }


## 2.3 代理主题

代理主题是"悟空"，它对应的类是WuKong。

    public class WuKong extends AbstractAppearance {
        AbstractAppearance missGao = new MissGao();

        @Override
        public void smile() {
            missGao.smile();        
            System.out.println("Actual, it's wukong smiles.");
        }
    }

 

## 2.4 客户端测试程序

客户端是"八戒"，它对应的类是BaJie。

    public class BaJie {
        public static void main(String[] args) {
            // 八戒只认识"高小姐的样貌"，而该"高小姐的样貌"实际上是悟空假扮的。
            AbstractAppearance appearance = new WuKong();
            appearance.smile();
        }
    }

 

运行结果：

    Miss Gao smiles.
    Actual, it's wukong smiles.

 
