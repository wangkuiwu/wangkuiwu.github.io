---
layout: post
title: "设计模式18之 观察者(Observer)模式(行为模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-28 09:01
---
 
> 本章介绍"观察者模式"。

> **目录**  
[1. 观察者模式简介](#anchor1)  
[2. Java中的观察者模式](#anchor2)  

 
<a name="anchor1"></a>
# 1. 观察者模式简介

　观察者模式是对象的行为模式，又叫发布-订阅(Publish/Subscribe)模式、模型-视图(Model/View)模式、源-监听器(Source/Listener)模式或从属者(Dependents)模式。
　
　观察者模式定义了一种一对多的依赖关系，让多个观察者对象同时监听某一个主题对象。这个主题对象在状态上发生变化时，会通知所有观察者对象，使它们能够自动更新自己。

　例如，马路上的交通指示灯为红灯亮的话，汽车会停止，行人可以通行。在这个场景中，交通指示等就是被观察者，而行人和汽车则是观察者，他们会根据观察到的交通指示灯的状况作出相应的行为。

　观察者的UML类图如下：

![img](/media/pic/design_patterns/pattern18_01.jpg)

　　观察者模式所涉及的角色有：**抽象主题(Subject)，具体主题(ConcreteSubject)，抽象观察者(Observer)，具体观察者(ConcreteObserver)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象主题 | 抽象主题角色把所有对观察者对象的引用保存在一个集合(比如Vector对象)里，每个主题都可以有任何数量的观察者。抽象主题提供一个接口，可以增加和删除观察者对象，抽象主题角色又叫做抽象被观察者(Observable)角色，一般用一个抽象类或者一个接口实现。 |
| 具体主题 | 将有关状态存入具体观察者对象；在具体主题的内部状态改变时，给所有登记过的观察者发出通知。具体主题角色又叫做具体被观察者(Concrete Observable)角色。具体主题通常用一个具体子类实现。 |
| 抽象观察者 | 为所有的具体观察者定义一个接口，在得到主题的通知时更新自己。这个接口叫做更新接口。抽象观察者一般用一个抽象类或者一个接口实现。 |
| 具体观察者 | 存储与主题的状态自恰的状态。具体观察者角色实现抽象观察者角色所要求的更新接口，以便使本身的状态与主题的状态相协调。如果需要，具体观察者角色可以保持一个指向具体主题对象的引用。它通常用一个具体子类实现。 |


## 示意代码

抽象主题类

	import java.util.Vector;
	import java.util.Enumeration;

	abstract public class Subject {
	 
		private Vector observersVector = new Vector();
		
		// 注册观察者对象
		public void attach(Observer observer) {
			observersVector.addElement(observer);
		}

		// 注销观察者对象
		public void detach(Observer observer) {
			observersVector.removeElement(observer);
		}

		// 通知所有注册的观察者对象
		public void notifyObservers() {
			Enumeration enu = observers();
			while (enu.hasMoreElements()) {
				((Observer)enu.nextElement()).update();
			}
		}

		public Enumeration observers(){
			return ((Vector)observersVector.clone()).elements();
		}
	}

具体主题类

	public class ConcreteSubject extends Subject{
		private String state;
		
		// 改变主题的方法。
		public void change(String newState){
			state = newState;
			this.notifyObservers();
		}
	}

抽象观察者类

	public interface Observer {
		//  通知接口
		public void update();
	}

具体观察者类

	public class ConcreteObserver implements Observer {
		
		@Override
		public void update() {
			System.out.println("I am notified.");
		}
	}

客户端类

	public class Client {

		private static ConcreteSubject subject;
		private static Observer observer;
		public static void main(String[] args) {
			// 创建主题
			subject = new ConcreteSubject();
			// 创建观察者
			observer = new ConcreteObserver();
			// 将观察者注册到"主题"上
			subject.attach(observer);
			// 改变主题对象的状态
			subject.change("new state");
		}
	}

运行结果：

	I am notified.

 
<a name="anchor2"></a>
# 2. Java中的观察者模式

　　Java语言本身支持"观察者模式"，它提供了相应的"抽象主题类(Observable)"和"抽象观察者类(Observer)"。

　　Observable是个实例类。它提供公开的方法支持观察者对象，这些方法中有两个对Observable的子类非常重要：一个是setChanged()，另一个是notifyObservers()。第一方法setChanged()被调用之后会设置一个内部标记变量，代表被观察者对象的状态发生了变化。第二个是notifyObservers()，这个方法被调用时，会调用所有登记过的观察者对象的update()方法，使这些观察者对象可以更新自己。

　　Observer是一个接口。它只定义了一个方法，即update()方法，当被观察者对象的状态发生变化时，被观察者对象的notifyObservers()方法就会调用这一方法。

　　下面演示Java中Observer和Observable的用法。

　　以公鸡打鸣来建模：太阳是被观察者，公鸡是观察者；当公鸡观察到太阳升起的时候，就打鸣。

## 2.1 太阳

太阳是被观察者，它继承了Observable。

	import java.util.Observable;

	public class Sun extends Observable {

		public void rise() {
			System.out.println("Sun rise.");
			// 设置"被观察者"的状态标记，表示它发生了变化。
			this.setChanged();
			// 通知"观察者"该变化。
			this.notifyObservers();
		}
	}

## 2.2 公鸡

公鸡是观察者，它实现了Observer对象。

	import java.util.Observer;
	import java.util.Observable;

	public class Cock implements Observer {
		private Sun sun;

		public Cock(Sun sun) {
			this.sun = sun;
			// 将观察者Cock注册到"被观察者sun"上。
			sun.addObserver(this);
		}

		// "被观察者"发生变化时，"观察者"对应的响应方法。
		@Override
		public void update(Observable o, Object arg) {
			System.out.println("Cock gogoda,gogoda,gogoda...");
		}
	}

## 2.3 客户端测试程序

	public class Client {

		private static Cock cock;
		private static Sun sun;
		public static void main(String[] args) {
			// 新建"太阳"(被观察者)
			sun = new Sun();
			// 新建"公鸡"(观察者)
			cock = new Cock(sun);
			// 太阳升起。公鸡观察到太阳升级后，会打鸣！
			sun.rise();
		}
	}

运行结果：

	Sun rise.
	Cock gogoda,gogoda,gogoda...

 
