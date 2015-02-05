---
layout: post
title: "Java 集合系列02之 Collection架构"
description: "java collection"
category: java
tags: [java]
date: 2012-02-02 09:01
---


本文，我们将对Collection进行概括。下面先看看Collection的一些框架类的关系图：

![img](/media/pic/java/collection/collection02.jpg)

Collection是一个接口，它主要的两个分支是：**List** 和 **Set**。

List和Set都是接口，它们继承于Collection。List是有序的队列，List中可以有重复的元素；而Set是数学概念中的集合，Set中没有重复元素！  
List和Set都有它们各自的实现类。

  为了方便实现，集合中定义了AbstractCollection抽象类，它实现了Collection中的绝大部分函数；这样，在Collection的实现类中，我们就可以通过继承AbstractCollection省去重复编码。AbstractList和AbstractSet都继承于AbstractCollection，具体的List实现类继承于AbstractList，而Set的实现类则继承于AbstractSet。

  另外，Collection中有一个iterator()函数，它的作用是返回一个Iterator接口。通常，我们通过Iterator迭代器来遍历集合。ListIterator是List接口所特有的，在List接口中，通过ListIterator()返回一个ListIterator对象。

  接下来，我们看看各个接口和抽象类的介绍；然后，再对实现类进行详细的了解。

> **目录**  
> **1**. [Collection简介](#anchor1)   
> **2**. [List简介](#anchor2)  
> **3**. [Set简介](#anchor3)  
> **4**. [AbstractCollection](#anchor4)  
> **5**. [AbstractList](#anchor5)  
> **6**. [AbstractSet](#anchor6)  
> **7**. [Iterator](#anchor7)  
> **8**. [ListIterator](#anchor8)  

 
<a name="anchor1"></a>
# 1. Collection简介

Collection的定义如下：

    public interface Collection<E> extends Iterable<E> {}

它是一个接口，是高度抽象出来的集合，它包含了集合的基本操作：添加、删除、清空、遍历(读取)、是否为空、获取大小、是否保护某元素等等。


Collection接口的所有子类(直接子类和间接子类)都必须实现2种构造函数：不带参数的构造函数 和 参数为Collection的构造函数。带参数的构造函数，可以用来转换Collection的类型。

    // Collection的API
    abstract boolean         add(E object)
    abstract boolean         addAll(Collection<? extends E> collection)
    abstract void            clear()
    abstract boolean         contains(Object object)
    abstract boolean         containsAll(Collection<?> collection)
    abstract boolean         equals(Object object)
    abstract int             hashCode()
    abstract boolean         isEmpty()
    abstract Iterator<E>     iterator()
    abstract boolean         remove(Object object)
    abstract boolean         removeAll(Collection<?> collection)
    abstract boolean         retainAll(Collection<?> collection)
    abstract int             size()
    abstract <T> T[]         toArray(T[] array)
    abstract Object[]        toArray()

 
<a name="anchor2"></a>
# 2. List简介

List的定义如下：

    public interface List<E> extends Collection<E> {}

List是一个继承于Collection的接口，即List是集合中的一种。List是有序的队列，List中的每一个元素都有一个索引；第一个元素的索引值是0，往后的元素的索引值依次+1。和Set不同，List中允许有重复的元素。
List的官方介绍如下：
> A List is a collection which maintains an ordering for its elements. Every element in the List has an index. Each element can thus be accessed by its index, with the first index being zero. Normally, Lists allow duplicate elements, as compared to Sets, where elements have to be unique.

 

关于API方面。既然List是继承于Collection接口，它自然就包含了Collection中的全部函数接口；由于List是有序队列，它也额外的有自己的API接口。主要有“添加、删除、获取、修改指定位置的元素”、“获取List中的子队列”等。

    // Collection的API
    abstract boolean         add(E object)
    abstract boolean         addAll(Collection<? extends E> collection)
    abstract void            clear()
    abstract boolean         contains(Object object)
    abstract boolean         containsAll(Collection<?> collection)
    abstract boolean         equals(Object object)
    abstract int             hashCode()
    abstract boolean         isEmpty()
    abstract Iterator<E>     iterator()
    abstract boolean         remove(Object object)
    abstract boolean         removeAll(Collection<?> collection)
    abstract boolean         retainAll(Collection<?> collection)
    abstract int             size()
    abstract <T> T[]         toArray(T[] array)
    abstract Object[]        toArray()
    // 相比与Collection，List新增的API：
    abstract void                add(int location, E object)
    abstract boolean             addAll(int location, Collection<? extends E> collection)
    abstract E                   get(int location)
    abstract int                 indexOf(Object object)
    abstract int                 lastIndexOf(Object object)
    abstract ListIterator<E>     listIterator(int location)
    abstract ListIterator<E>     listIterator()
    abstract E                   remove(int location)
    abstract E                   set(int location, E object)
    abstract List<E>             subList(int start, int end)

 
<a name="anchor3"></a>
# 3. Set简介

Set的定义如下：

    public interface Set<E> extends Collection<E> {}

Set是一个继承于Collection的接口，即Set也是集合中的一种。Set是没有重复元素的集合。

关于API方面。Set的API和Collection完全一样。

    // Set的API
    abstract boolean         add(E object)
    abstract boolean         addAll(Collection<? extends E> collection)
    abstract void             clear()
    abstract boolean         contains(Object object)
    abstract boolean         containsAll(Collection<?> collection)
    abstract boolean         equals(Object object)
    abstract int             hashCode()
    abstract boolean         isEmpty()
    abstract Iterator<E>     iterator()
    abstract boolean         remove(Object object)
    abstract boolean         removeAll(Collection<?> collection)
    abstract boolean         retainAll(Collection<?> collection)
    abstract int             size()
    abstract <T> T[]         toArray(T[] array)
    abstract Object[]         toArray()

 
<a name="anchor4"></a>
# 4. AbstractCollection

AbstractCollection的定义如下：

    public abstract class AbstractCollection<E> implements Collection<E> {}

AbstractCollection是一个抽象类，它实现了Collection中除iterator()和size()之外的函数。  
AbstractCollection的主要作用：它实现了Collection接口中的大部分函数。从而方便其它类实现Collection，比如ArrayList、LinkedList等，它们这些类想要实现Collection接口，通过继承AbstractCollection就已经实现了大部分的接口了。

 
<a name="anchor5"></a>
# 5. AbstractList

AbstractList的定义如下：

    public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {}

AbstractList是一个继承于AbstractCollection，并且实现List接口的抽象类。它实现了List中除size()、get(int location)之外的函数。  
AbstractList的主要作用：它实现了List接口中的大部分函数。从而方便其它类继承List。  
另外，和AbstractCollection相比，AbstractList抽象类中，实现了iterator()接口。

 
<a name="anchor6"></a>
# 6. AbstractSet

AbstractSet的定义如下： 

    public abstract class AbstractSet<E> extends AbstractCollection<E> implements Set<E> {}

AbstractSet是一个继承于AbstractCollection，并且实现Set接口的抽象类。由于Set接口和Collection接口中的API完全一样，Set也就没有自己单独的API。和AbstractCollection一样，它实现了List中除iterator()和size()之外的函数。  
AbstractSet的主要作用：它实现了Set接口中的大部分函数。从而方便其它类实现Set接口。

 
<a name="anchor7"></a>
# 7. Iterator

Iterator的定义如下：

    public interface Iterator<E> {}

Iterator是一个接口，它是集合的迭代器。集合可以通过Iterator去遍历集合中的元素。Iterator提供的API接口，包括：是否存在下一个元素、获取下一个元素、删除当前元素。  
**注意**：Iterator遍历Collection时，是fail-fast机制的。即，当某一个线程A通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程A访问集合时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。关于fail-fast的详细内容，我们会在[fail-fast总结][link_java_collection_04]后面专门进行说明。

    // Iterator的API
    abstract boolean hasNext()
    abstract E next()
    abstract void remove()

 
<a name="anchor8"></a>
# 8. ListIterator

ListIterator的定义如下：

    public interface ListIterator<E> extends Iterator<E> {}

ListIterator是一个继承于Iterator的接口，它是队列迭代器。专门用于便利List，能提供向前/向后遍历。相比于Iterator，它新增了添加、是否存在上一个元素、获取上一个元素等等API接口。

    // ListIterator的API
    // 继承于Iterator的接口
    abstract boolean hasNext()
    abstract E next()
    abstract void remove()
    // 新增API接口
    abstract void add(E object)
    abstract boolean hasPrevious()
    abstract int nextIndex()
    abstract E previous()
    abstract int previousIndex()
    abstract void set(E object)


# 更多内容

[00. Java 集合系列目录(Category)][link_java_collection_00]  
[01. Java 集合系列01之 总体框架][link_java_collection_01]  
[02. Java 集合系列02之 Collection架构][link_java_collection_02]  
[03. Java 集合系列03之 ArrayList详细介绍(源码解析)和使用示例][link_java_collection_03]  
[04. Java 集合系列04之 fail-fast总结(通过ArrayList来说明fail-fast的原理、解决办法)][link_java_collection_04]  
[05. Java 集合系列05之 LinkedList详细介绍(源码解析)和使用示例][link_java_collection_05]  
[06. Java 集合系列06之 Vector详细介绍(源码解析)和使用示例][link_java_collection_06]  
[07. Java 集合系列07之 Stack详细介绍(源码解析)和使用示例][link_java_collection_07]  
[08. Java 集合系列08之 List总结(LinkedList, ArrayList等使用场景和性能分析)][link_java_collection_08]  
[09. Java 集合系列09之 Map架构][link_java_collection_09]  
[10. Java 集合系列10之 HashMap详细介绍(源码解析)和使用示例][link_java_collection_10]  
[11. Java 集合系列11之 Hashtable详细介绍(源码解析)和使用示例][link_java_collection_11]  
[12. Java 集合系列12之 TreeMap详细介绍(源码解析)和使用示例][link_java_collection_12]  
[13. Java 集合系列13之 WeakHashMap详细介绍(源码解析)和使用示例][link_java_collection_13]  
[14. Java 集合系列14之 Map总结(HashMap, Hashtable, TreeMap, WeakHashMap等使用场景)][link_java_collection_14]  
[15. Java 集合系列15之 Set架构][link_java_collection_15]  
[16. Java 集合系列16之 HashSet详细介绍(源码解析)和使用示例][link_java_collection_16]  
[17. Java 集合系列17之 TreeSet详细介绍(源码解析)和使用示例][link_java_collection_17]  
[18. Java 集合系列18之 Iterator和Enumeration比较][link_java_collection_18]

[link_java_collection_00]: /2012/02/01/collection-00-index
[link_java_collection_01]: /2012/02/01/collection-01-summary
[link_java_collection_02]: /2012/02/02/collection-02-framework
[link_java_collection_03]: /2012/02/03/collection-03-arraylist
[link_java_collection_04]: /2012/02/04/collection-04-fail-fast
[link_java_collection_05]: /2012/02/05/collection-05-linkedlist
[link_java_collection_06]: /2012/02/06/collection-06-vector
[link_java_collection_07]: /2012/02/07/collection-07-stack
[link_java_collection_08]: /2012/02/08/collection-08-List
[link_java_collection_09]: /2012/02/09/collection-09-map
[link_java_collection_10]: /2012/02/10/collection-10-hashmap
[link_java_collection_11]: /2012/02/11/collection-11-hashtable
[link_java_collection_12]: /2012/02/12/collection-12-treemap
[link_java_collection_13]: /2012/02/13/collection-13-weakhashmap
[link_java_collection_14]: /2012/02/14/collection-14-mapsummary
[link_java_collection_15]: /2012/02/15/collection-15-set
[link_java_collection_16]: /2012/02/16/collection-16-hashset
[link_java_collection_17]: /2012/02/17/collection-17-treeset
[link_java_collection_18]: /2012/02/18/collection-18-iterator_enumeration
