---
layout: post
title: "设计模式19之 迭代器(Iterator)模式(行为模式)"
description: "java"
category: pattern
tags: [java, pattern]
date: 2012-10-29 09:01
---
 

> 本章介绍"迭代器模式"。

> **目录**  
[1. 迭代器模式简介](#anchor1)  
[2. 外禀迭代器](#anchor2)  
[3. 内禀迭代器](#anchor3)  

 
<a name="anchor1"></a>
# 1. 迭代器模式简介

　　迭代器模式又叫游标(Cursor)模式，是对象的行为模式。迭代器模式可以顺序地访问一个聚集中的元素而不必暴露聚集的内部表象（internal representation）。

　　迭代器模式是设计模式中最常见的几个模式之一。在Java的集合(Collection)框架中，广泛的使用迭代器(Iterator和Enumeration)来遍历集合的元素。

迭代器模式涉及到以下几个角色：**抽象迭代器(Iterator)，具体迭代器(ConcreteIterator)，聚集(Aggregate)，具体聚集(ConcreteAggregate)，客户端(Client)**。

|     角色   |       说明      |
| ---------- | --------------- |
| 抽象迭代器 | 此抽象角色定义出遍历元素所需的接口。 |
| 具体迭代器 | 此角色实现了抽象迭代器的接口，并保持迭代过程中的游标位置。 |
| 聚集 | 此抽象角色给出创建迭代器(Iterator)对象的接口。 |
| 具体聚集 | 实现了创建迭代器(Iterator)对象的接口，返回一个合适的具体迭代器实例。 |
| 客户端 | 持有对聚集及其迭代器对象的引用，调用迭代器对象的迭代接口，也有可能通过迭代器操作聚集元素的增加和删除。 |

<br/>

　　迭代器模式有两种：**外禀迭代器**和**内禀迭代器**。  
外禀迭代器 -- "具体迭代器"是在"具体聚集"之外实现的。  
内禀迭代器 -- "具体迭代器"是在"具体聚集"里面实现的，"具体迭代器"是"具体聚集"的私有内部类。

　　在外禀迭代器中，"具体聚集"向外提供了访问聚集中各个元素的接口；而在内禀迭代器中，"具体聚集"包含了内部类"具体迭代器"，这就意味着"具体迭代器"可以直接访问"具体聚集"的成员对象，而不需要通过函数接口去访问。

 
<a name="anchor2"></a>
# 2. 外禀迭代器

　　如果迭代器是在聚集结构之外实现的，这样的迭代器被称为外禀迭代器(Extrinsic Iterator)。

下面看看"外禀迭代器"中各个角色的代码。

## 2.1 抽象迭代器类

	public interface Iterator {
		// 迭代方法：移动到第一个元素
		public void first();

		// 迭代方法：移动到下一个元素
		public void next();

		// 迭代方法：是否为最后一个元素
		public boolean isDone();

		// 迭代方法：返还当前元素
		public Object currentItem();
	}

## 2.2 具体迭代器类

	public class ConcreteIterator implements Iterator {
		private ConcreteAggregate agg;
		// 索引位置
		private int index = 0;
		// 集合大小
		private int size = 0;
		
		public ConcreteIterator(ConcreteAggregate agg){
			this.agg = agg;
			this.size = agg.size();
			index = 0;
		}

		// 迭代方法：移动到第一个元素
		@Override
		public void first() {
			index = 0;
		}

		// 迭代方法：是否为最后一个元素
		@Override
		public boolean isDone() {
			return (index >= size);
		}

		// 迭代方法：移动到下一个元素
		@Override
		public void next() {
			if(index < size) {
				index ++;
			}
		}
	 
		// 迭代方法：返还当前元素
		@Override
		public Object currentItem() {
			return agg.getElement(index);
		}
	}

## 2.3 聚集类

	abstract public class Aggregate {
		// 工厂方法：返回迭代器对象
		public abstract Iterator createIterator();
	}

## 2.4 具体聚集类

	public class ConcreteAggregate extends Aggregate {
		
		private Object[] objs = {"Monk Tang",
			"Monkey", "Pigsy",
			"Sandy", "Horse"};
		
		@Override
		public Iterator createIterator() {
			return new ConcreteIterator(this);
		}

		// 取值方法：向外界提供聚集元素
		public Object getElement(int index){
			
			if(index < objs.length){
				return objs[index];
			}else{
				return null;
			}
		}

		// 取值方法：向外界提供聚集的大小
		public int size(){
			return objs.length;
		}
	}

## 2.5 客户端类

	public class Client {

		private Iterator it;
		private Aggregate agg = new ConcreteAggregate();
		public void operation() {
			it = agg.createIterator();
			while(!it.isDone()) {
				System.out.println(it.currentItem());
				it.next();
			}
		}

		public static void main(String[] args) {
			Client client = new Client();
			client.operation();
		}

	}

运行结果：

	Monk Tang
	Monkey
	Pigsy
	Sandy
	Horse

 



<a name="anchor3"></a>
# 3. 内禀迭代器

　　如果将"外禀迭代器"中的"具体迭代器"改写成"具体聚集类"的一个私有类，即迭代器是在聚集结构之内实现；这样的迭代器，就被称为内禀迭代器(Intrinsic Iterator)。

下面看看"内禀迭代器"中各个角色的代码。

## 3.1 抽象迭代器类

	public interface Iterator {
		// 迭代方法：移动到第一个元素
		public void first();

		// 迭代方法：移动到下一个元素
		public void next();

		// 迭代方法：是否为最后一个元素
		public boolean isDone();

		// 迭代方法：返还当前元素
		public Object currentItem();
	}

## 3.2 聚集类

	abstract public class Aggregate {
		// 工厂方法：返回迭代器对象
		public abstract Iterator createIterator();
	}

## 3.3 具体聚集类

	public class ConcreteAggregate extends Aggregate {
		
		private Object[] objs = {"Monk Tang",
			"Monkey", "Pigsy",
			"Sandy", "Horse"};
		
		@Override
		public Iterator createIterator() {
			return new ConcreteIterator();
		}

		private class ConcreteIterator 
				implements Iterator {
			// 索引位置
			private int index = 0;
			
			// 迭代方法：移动到第一个元素
			@Override
			public void first() {
				index = 0;
			}

			// 迭代方法：是否为最后一个元素
			@Override
			public boolean isDone() {
				return (index == objs.length);
			}

			// 迭代方法：移动到下一个元素
			@Override
			public void next() {
				if(index < objs.length) {
					index ++;
				}
			}
		 
			// 迭代方法：返还当前元素
			@Override
			public Object currentItem() {
				return objs[index];
			}
		}
	}

## 3.4 客户端类

	public class Client {

		private Iterator it;
		private Aggregate agg = new ConcreteAggregate();
		public void operation() {
			it = agg.createIterator();
			while(!it.isDone()) {
				System.out.println(it.currentItem());
				it.next();
			}
		}

		public static void main(String[] args) {
			Client client = new Client();
			client.operation();
		}

	}

运行结果：

	Monk Tang
	Monkey
	Pigsy
	Sandy
	Horse

