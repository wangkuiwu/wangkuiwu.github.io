---
layout: post
title: "Java 集合系列01之 总体框架"
description: "java collection"
category: java
tags: [java]
date: 2012-02-01 09:01
---


> Java集合是java提供的工具包，包含了常用的数据结构：集合、链表、队列、栈、数组、映射等。Java集合工具包位置是java.util.\*   
Java集合主要可以划分为4个部分：List列表、Set集合、Map映射、工具类(Iterator迭代器、Enumeration枚举类、Arrays和Collections)。  

Java集合工具包框架如下图。

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/pictures/java/collections/collection01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/pictures/java/collections/collection01.jpg" alt="" /></a>

说明：看上面的框架图，先抓住它的主干，即Collection和Map。

1. Collection是一个接口，是高度抽象出来的集合，它包含了集合的基本操作和属性。

  Collection包含了List和Set两大分支。

  (01) List是一个有序的队列，每一个元素都有它的索引。第一个元素的索引值是0。  
          List的实现类有LinkedList, ArrayList, Vector, Stack。

  (02) Set是一个不允许有重复元素的集合。  
          Set的实现类有HastSet和TreeSet。HashSet依赖于HashMap，它实际上是通过HashMap实现的；TreeSet依赖于TreeMap，它实际上是通过TreeMap实现的。

2. Map是一个映射接口，即key-value键值对。Map中的每一个元素包含“一个key”和“key对应的value”。

   AbstractMap是个抽象类，它实现了Map接口中的大部分API。而HashMap，TreeMap，WeakHashMap都是继承于AbstractMap。  
   Hashtable虽然继承于Dictionary，但它实现了Map接口。

接下来，再看Iterator。它是遍历集合的工具，即我们通常通过Iterator迭代器来遍历集合。我们说Collection依赖于Iterator，是因为Collection的实现类都要实现iterator()函数，返回一个Iterator对象。  
ListIterator是专门为遍历List而存在的。

再看Enumeration，它是JDK 1.0引入的抽象类。作用和Iterator一样，也是遍历集合；但是Enumeration的功能要比Iterator少。在上面的框图中，Enumeration只能在Hashtable, Vector, Stack中使用。

最后，看Arrays和Collections。它们是操作数组、集合的两个工具类。

有了上面的整体框架之后，我们接下来对每个类分别进行分析。

