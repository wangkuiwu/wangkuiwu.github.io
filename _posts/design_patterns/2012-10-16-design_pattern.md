---
layout: post
title: "设计模式06之 建造模式(创建模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-16 09:01
---
 
> 本章介绍"建造模式"。

> **目录**  
[1. 建造模式简介](#anchor1)  
[2. 建造模式代码模型](#anchor2)  
[3. 建造模式示例](#anchor3)  

 
<a name="anchor1"></a>
# 1. 建造模式简介

建造模式是"创建模式"(创建对象的模式)。当我们要创建的对象比较复杂的时候，可以将对象的"创建"和"表示"分割开来；从而可以使一个构造过程创建具有不同表示的产品。这种参见对象的方式就是建造模式。

**建造模式的使用场景**

(01), 对象包含复杂的内部结构。每个内部成分本身可以是对象，也可以仅仅是一个对象的一个组成成分。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，一个电子邮件由发件人地址、收件人地址、主题、内容、附件等内容。我们可以根据这些组成部分组合成一份邮件，然后发送出去。

(02), 对象的各个属性相互依赖。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，某对象包含许多性质，而这些性质必须按照某个顺序赋值才有意义。

(03), 对象创建过程会使用到系统中的其他对象，而这些对象在产品对象的创建过程中不易得到。

 

下面看看它的结构图：

![img](/media/pic/design_patterns/pattern06_01.jpg)

建造模式包括4个组成部分：**导演者(Director), 抽象建造者(Builder), 具体建造者(ConcreteBuilder) 和 产品(Product)**。

|     组成部分   |       说明      |
| -------------- | --------------- |
| Director | 它会调用具体创建者来创造一个产品对象。 |
| Builder | 它会给出"创建一个产品对象的各个组成成分"的抽象接口。 |
| ConcreteBuilder | 实现"抽象建造者"给出的抽象接口。 |
| Product | 要被创建的产品对象。 |

 


<a name="anchor2"></a>
# 2. 建造模式代码模型

    public class Director {
        private Builder builder;

        // 构造产品
        public void construct() {
            builder = new ConcreteBuilder();
            builder.buildPart1();
            builder.buildPart2();
        }
    }

    abstract public class Builder {
        // 构造"产品的Part1"的接口
        public abstract void buildPart1();
        // 构造"产品的Part2"的接口
        public abstract void buildPart2();
        // 返回产品
        public abstract Product retrieveResult();
    }

    public class ConcreteBuilder {
        private Product product = new Product();
        public void buildPart1() {
            // 参见产品的Part1
        }
        public void buildPart2() {
            // 参见产品的Part2
        }
        public Product retrieveResult() {
            return product;
        }
    }

    public class Product {
         // 产品组成部分...
    }

 
<a name="anchor3"></a>
# 3. 建造模式示例

假设由一个电子杂志系统，定期的向用户的发送电子邮箱。发送的电子邮件模板由两种：欢迎和欢送。邮件包括"发送者地址"，"接收者地址"，“主题”，“内容”，“发送日期”等内容；其中，“发送者地址”和“接收者地址”是必填的。

下面通过建造模式实现该系统，对应的UML图如下：

![img](/media/pic/design_patterns/pattern06_02.jpg)

建造模式包括4个组成部分："导演者", "抽象建造者", "具体建造者"和"产品"这4个角色分别如下：

## 3.1 "导演者"

导演者是Director。Director会根据Builder对象，构造出邮件。

Director的源码

    public class Director {

        private Builder builder;

        public Director(Builder builder) {
            this.builder = builder;
        }

        public void construct(String from, String to) {
            builder.buildFrom(from);
            builder.buildTo(to);
            builder.buildSubject();
            builder.buildBody();
            builder.buildSendDate();

            builder.send();
        }
    }

 

## 3.2 "抽象建造者"

抽象建造者是Builder。Builder是抽象类，它包含具体的邮件对象msg，msg是在Builder的子类中根据所要创建的消息来初始化的。此外，Builder提供了构造邮件的函数接口。

Builder的源码

    import java.util.Date;

    abstract public class Builder {

        protected AutoMessage msg;

        public Builder() {
        }

        public abstract void buildBody() ;
        public abstract void buildSubject() ;

        public void buildFrom(String from) {
            msg.setFrom(from);
        }
        public void buildTo(String to) {
            msg.setTo(to);
        }
        public void buildSendDate() {
            msg.setSendDate(new Date());
        }
        public void send() {
            msg.send();
        }
    }

 

## 3.3 "具体建造者"

具体建造者是WelcomeBuilder和GoodbyeBuilder。WelcomeBuilder是欢迎消息，GoodbyeBuilder是欢送消息。

WelcomeBuilder的源码

    public class WelcomeBuilder extends Builder {

        public WelcomeBuilder() {
            System.out.println("create WelcomeBuilder");
            this.msg = new WelcomeMessage();
        }

        public void buildBody() {
            msg.setBody("Welcome body!");
        }
        public void buildSubject() {
            msg.setSubject("Welcome subject!");
        }
    }

GoodbyeBuilder的源码

    public class GoodbyeBuilder extends Builder {

        public GoodbyeBuilder() {
            System.out.println("create GoodbyeBuilder");
            msg = new GoodbyeMessage();
        }

        public void buildBody() {
            msg.setBody("Goodbye body!");
        }
        public void buildSubject() {
            msg.setSubject("Goodbye subject!");
        }
    }

 

## 3.4 "产品"

产品是AutoMessage。AutoMessage是抽象出来的产品类，它包含两个子类WelcomeBuilder和GoodbyeMessage。

AutoMessage的源码

    import java.util.Date;

    abstract public class AutoMessage {

        protected String from;
        protected String to;
        protected String subject;
        protected String body;
        protected Date sendDate;

        public void setFrom(String from) {
            this.from = from;
        }
        public String getFrom() {
            return from;
        }
        public void setTo(String to) {
            this.to = to;
        }
        public String getTo() {
            return to;
        }
        public void setSubject(String subject) {
            this.subject = subject;
        }
        public String getSubject() {
            return subject;
        }
        public void setBody(String body) {
            this.body = body;
        }
        public String getBody() {
            return body;
        }
        public void setSendDate(Date sendDate) {
            this.sendDate = sendDate;
        }
        public Date getSendDate() {
            return sendDate;
        }
        public void send() {
            System.out.println("send MSG[Subject: \""+subject+"\", Body: \""+body+"\"] from \""+from+"\" to \""+to+"\" @date:\""+sendDate+"\"");
        }
    }

WelcomeMessage的源码

    public class WelcomeMessage extends AutoMessage {

        public WelcomeMessage() {
            System.out.println("Create WelcomeMessage");
        }

        public void sayWelcome() {
            System.out.println("Welcome.");
        }
    }

GoodbyeMessage的源码

    public class GoodbyeMessage extends AutoMessage {

        public GoodbyeMessage() {
            System.out.println("Create GoodbyeMessage");
        }

        public void sayGoodbye() {
            System.out.println("Goodbye.");
        }
    }

 

## 2.5 客户端测试程序

客户端是"Client"。

Client的源码

    public class Client {

        private static Director dw, dg;
        private static Builder bw, bg;

        public static void main(String[] args) {
            bw = new WelcomeBuilder();
            dw = new Director(bw);
            dw.construct("skywang12345@gmail.com", "java_patterns@gmail.com");

            bg = new GoodbyeBuilder();
            dg = new Director(bg);
            dg.construct("skywang54321@gmail.com", "java_patterns@gmail.com");
        }
    }

运行结果：

    create WelcomeBuilder
    Create WelcomeMessage
    send MSG[Subject: "Welcome subject!", Body: "Welcome body!"] from "skywang12345@gmail.com" to "java_patterns@gmail.com" @date:"Sun Jan 19 10:48:02 CST 2014"
    create GoodbyeBuilder
    Create GoodbyeMessage
    send MSG[Subject: "Goodbye subject!", Body: "Goodbye body!"] from "skywang54321@gmail.com" to "java_patterns@gmail.com" @date:"Sun Jan 19 10:48:02 CST 2014"

 
