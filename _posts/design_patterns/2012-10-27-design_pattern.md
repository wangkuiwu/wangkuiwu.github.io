---
layout: post
title: "设计模式17之 模板方法(Template Method)模式(行为模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-27 09:01
---
 
> 本章介绍"模板方法模式"。

> **目录**  
[1. 模板模式简介](#anchor1)  
[2. 模板方法模式示例](#anchor2)  

 
<a name="anchor1"></a>
# 1. 模板模式简介

　　模板方法(Template Method)模式是类的行为模式。准备一个抽象类，将部分逻辑以具体方法以及具体构造函数的形式实现，然后声明一些抽象方法来迫使子类实现剩余的逻辑。不同的子类可以以不同的方式实现这些抽象方法，从而对剩余的逻辑有不同的实现。这就是模板方法模式的用意。

　　模板方法模式是所有模式中最为常见的几个模式之一，而且很可能我们自己使用过模板方法模式而没有意识到自己已经使用了这个模式。模板方法模式是基于继承的代码复用的基本技术。

　　Java的集合就是一个典型的，利用了模板方法模式的例子。Java集合中的Collection集合包括List和Set两大组成部分。List是队列，而Set是没有重复元素的集合。它们共同的接口都在Collection接口声明；例如，都包含了size()，isEmpty()方法。而AbstractCollection这个抽象类则实现了它们共同的方法，其余未实现的方法定义为抽象方法。List和Set的实例类，就是通过继承AbstractCollection(或它的子类)，省去了许多重复性编码的工作！

<br/>
模板方法模式的UML类图：

![img](/media/pic/design_patterns/pattern17_01.jpg)

这里涉及到两个角色：**抽象模板(Abstract Template)，具体模板(Concrete Template)**。

  抽象模板有如下责任：  
• 定义了一个或多个抽象操作，以便让子类实现。这些抽象操作叫做基本操作，它们是一个顶级逻辑的组成步骤。  
• 定义并实现了一个模板方法。这个模板方法一般是一个具体方法，它给出了一个顶级逻辑的骨架，而逻辑的组成步骤在相应的抽象操作中，推迟到子类实现。顶级逻辑也有可能调用一些具体方法。

  具体模板有如下责任：  
• 实现父类所定义的一个或多个抽象方法，它们是一个顶级逻辑的组成步骤。  
• 每一个抽象模板角色都可以有任意多个具体模板角色与之对应，而每一个具体模板角色都可以给出这些抽象方法（也就是顶级逻辑的组成步骤）的不同实现，从而使得顶级逻辑的实现各不相同。



示意代码


	abstract public class AbstractClass {
		// 模板方法
		public void templateMethod(){
			hookMethod(); //调用基本方法(由子类实现)
			abstractMethod(); //调用基本方法(由子类实现)
			concreteMethod(); //调用基本方法(已经实现)
		}

		// 基本方法的声明（由子类实现，但抽象模板给出了默认实现）
		public void hookMethod() {}

		// 基本方法的声明（由子类实现）
		public abstract void abstractMethod();

		// 基本方法（已经实现）
		public final void concreteMethod(){
			// do something
		}
	}

	public class ConcreteClass extends AbstractClass {
		// 基本方法的实现
		@Override
		public void hookMethod() {
			// do something
		}

		// 基本方法的实现
		@Override
		public void abstractMethod() {
			// do something
		}
	}

 

　　抽象模板角色AbstractTemplate提供了两个具体方法templateMethod()和concreteMethod()；声明了2个抽象方法hookMethod()和abstractMethod()。

　　具体模板角色ConcreteTemplate实现了父类声明的基本方法hookMethod()和abstractMethod()。

 

<br/>
模板方法模式中的方法

模板方法中的方法可以分为两大类：**模板方法** 和 **基本方法**。

**模板方法**

一个模板方法是定义在抽象类中的，把基本操作方法组合在一起形成一个总算法或一个总行为的方法。  
一个抽象类可以有任意多个模板方法，而不限于一个。每一个模板方法都可以调用任意多个具体方法。

**基本方法**

基本方法又可以分为三种：**抽象方法(Abstract Method)、具体方法(Concrete Method)和钩子方法(Hook Method)**。  
• 抽象方法: 一个抽象方法由抽象类声明，由具体子类实现。在Java语言里抽象方法以abstract关键字标示。  
• 具体方法: 一个具体方法由抽象类声明并实现，而子类并不实现或置换。  
• 钩子方法: 一个钩子方法由抽象类声明并实现，而子类会加以扩展。通常抽象类给出的实现是一个空实现，作为方法的默认实现。

　　在前面的UML示例中，AbstractClass是一个抽象类，它带有三个方法。其中abstractMethod()是一个抽象方法，它由抽象类声明为抽象方法，并由子类实现；hookMethod()是一个钩子方法，它由抽象类声明并提供默认实现，并且由子类置换掉。concreteMethod()是一个具体方法，它由抽象类声明并实现。

提示：钩子方法的名字应当以do开始。

 
<a name="anchor2"></a>
# 2. 模板方法模式示例

　　考虑一个计算存款利息的例子。假设系统需要支持两种存款账号，即货币市场(Money Market)账号和定期存款(Certificate of Deposite)账号。这两种账号的存款利息是不同的，因此，在计算一个存户的存款利息额时，必须区分两种不同的账号类型。

　　这个系统的总行为应当是计算出利息，这也就决定了作为一个模板方法模式的顶级逻辑应当是利息计算。由于利息计算涉及到两个步骤：一个基本方法给出账号种类，另一个基本方法给出利息百分比。这两个基本方法构成具体逻辑，因为账号的类型不同，所以具体逻辑会有所不同。

　　显然，系统需要一个抽象角色给出顶级行为的实现，而将两个作为细节步骤的基本方法留给具体子类实现。由于需要考虑的账号有两种：一是货币市场账号，二是定期存款账号。系统的UML类图如下图所示。

## 2.1 抽象模板类

	abstract public class Account {

		protected String accountNumber;

		public Account() {
			accountNumber = null;
		}

		public Account(String accountNumber) {
			this.accountNumber = accountNumber;
		}

		// 模板方法，计算利息数额
		public final double calculateInterest(){
			double interestRate = doCalculateInterestRate();
			String accountType = doCalculateAccountType();
			double amount = calculateAmount(accountType, accountNumber);
			return amount * interestRate;
		}

		// 基本方法留给子类实现
		protected abstract String doCalculateAccountType();

		// 基本方法留给子类实现
		protected abstract double doCalculateInterestRate();

		// 基本方法，已经实现
		private double calculateAmount(String accountType, String accountNumber){
			// retrive amount from database
			return 7243.00D;
		}
	}

## 2.2 具体模板类

MoneyMarketAccount

	public class MoneyMarketAccount extends Account {

		@Override
		protected String doCalculateAccountType() {
			return "Money Market";
		}

		@Override
		protected double doCalculateInterestRate() {
			return 0.045D;
		}
	}

CDAccount

	public class CDAccount extends Account {

		@Override
		protected String doCalculateAccountType() {
			return "Certificate of Deposite";
		}

		@Override
		protected double doCalculateInterestRate() {
			return 0.065D;
		}
	}

## 2.3 客户端测试程序

	public class Client {

		private static Account acct = null;
		public static void main(String[] args) {
			acct = new MoneyMarketAccount();
			System.out.println("Interest from Money Market account: " + acct.calculateInterest());
			acct = new CDAccount();
			System.out.println("Interest from CD account: " + acct.calculateInterest());
		}
	}

运行结果：

	Interest from Money Market account: 325.935
	Interest from CD account: 470.795

结果说明：Account是抽象模板类，calculateInterest()是模板方法。不管是"货币市场帐号"或者"定期存款"，它们的利息计算方式都是通过抽象模板类的calculateInterest()来计算的；只不过涉及到的本金和利率不同而已。

 
