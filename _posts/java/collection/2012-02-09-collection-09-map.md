---
layout: post
title: "Java 集合系列09之 Map架构"
description: "java collection"
category: java
tags: [java]
date: 2012-02-09 09:01
---


> 前面，我们已经系统的对List进行了学习。接下来，我们先学习Map，然后再学习Set；因为Set的实现类都是基于Map来实现的(如，HashSet是通过HashMap实现的，TreeSet是通过TreeMap实现的)。

首先，我们看看Map架构。

![img](/media/pic/java/collection/collection09.jpg)

如上图：

(01) Map 是映射接口，Map中存储的内容是键值对(key-value)。  
(02) AbstractMap 是继承于Map的抽象类，它实现了Map中的大部分API。其它Map的实现类可以通过继承AbstractMap来减少重复编码。  
(03) SortedMap 是继承于Map的接口。SortedMap中的内容是排序的键值对，排序的方法是通过比较器(Comparator)。  
(04) NavigableMap 是继承于SortedMap的接口。相比于SortedMap，NavigableMap有一系列的导航方法；如"获取大于/等于某对象的键值对"、“获取小于/等于某对象的键值对”等等。  
(05) TreeMap 继承于AbstractMap，且实现了NavigableMap接口；因此，TreeMap中的内容是“有序的键值对”！  
(06) HashMap 继承于AbstractMap，但没实现NavigableMap接口；因此，HashMap的内容是“键值对，但不保证次序”！  
(07) Hashtable 虽然不是继承于AbstractMap，但它继承于Dictionary(Dictionary也是键值对的接口)，而且也实现Map接口；因此，Hashtable的内容也是“键值对，也不保证次序”。但和HashMap相比，Hashtable是线程安全的，而且它支持通过Enumeration去遍历。  
(08) WeakHashMap 继承于AbstractMap。它和HashMap的键类型不同，WeakHashMap的键是“弱键”。  

 

在对各个实现类进行详细之前，先来看看各个接口和抽象类的大致介绍。内容包括：  
> [1 Map](#anchor1)   
> [2 Map.Entry](#anchor2)   
> [3 AbstractMap](#anchor3)   
> [4 SortedMap](#anchor4)   
> [5 NavigableMap](#anchor5)   
> [6 Dictionary](#anchor6)   

 
<a name="anchor1"></a>
# 1. Map

Map的定义如下：

    public interface Map<K,V> { }

Map 是一个键值对(key-value)映射接口。Map映射中不能包含重复的键；每个键最多只能映射到一个值。  
Map 接口提供三种collection 视图，允许以键集、值集或键-值映射关系集的形式查看某个映射的内容。  
Map 映射顺序。有些实现类，可以明确保证其顺序，如 TreeMap；另一些映射实现则不保证顺序，如 HashMap 类。  
Map 的实现类应该提供2个“标准的”构造方法：第一个，void（无参数）构造方法，用于创建空映射；第二个，带有单个 Map 类型参数的构造方法，用于创建一个与其参数具有相同键-值映射关系的新映射。实际上，后一个构造方法允许用户复制任意映射，生成所需类的一个等价映射。尽管无法强制执行此建议（因为接口不能包含构造方法），但是 JDK 中所有通用的映射实现都遵从它。


**Map的API**

    abstract void                 clear()
    abstract boolean              containsKey(Object key)
    abstract boolean              containsValue(Object value)
    abstract Set<Entry<K, V>>     entrySet()
    abstract boolean              equals(Object object)
    abstract V                    get(Object key)
    abstract int                  hashCode()
    abstract boolean              isEmpty()
    abstract Set<K>               keySet()
    abstract V                    put(K key, V value)
    abstract void                 putAll(Map<? extends K, ? extends V> map)
    abstract V                    remove(Object key)
    abstract int                  size()
    abstract Collection<V>        values()

说明：  
(01) Map提供接口分别用于返回 键集、值集或键-值映射关系集。  
> entrySet()用于返回键-值集的Set集合  
> keySet()用于返回键集的Set集合  
> values()用户返回值集的Collection集合  
> 因为Map中不能包含重复的键；每个键最多只能映射到一个值。所以，键-值集、键集都是Set，值集时Collection。

(02) Map提供了“键-值对”、“根据键获取值”、“删除键”、“获取容量大小”等方法。

 
<a name="anchor2"></a>
# 2. Map.Entry

Map.Entry的定义如下：

    interface Entry<K,V> { }

Map.Entry是Map中内部的一个接口，Map.Entry是键值对，Map通过 entrySet() 获取Map.Entry的键值对集合，从而通过该集合实现对键值对的操作。

**Map.Entry的API**

    abstract boolean     equals(Object object)
    abstract K             getKey()
    abstract V             getValue()
    abstract int         hashCode()
    abstract V             setValue(V object)

 
<a name="anchor3"></a>
# 3. AbstractMap

AbstractMap的定义如下：

    public abstract class AbstractMap<K,V> implements Map<K,V> {}

AbstractMap类提供 Map 接口的骨干实现，以最大限度地减少实现此接口所需的工作。  
要实现不可修改的映射，编程人员只需扩展此类并提供 entrySet 方法的实现即可，该方法将返回映射的映射关系 set 视图。通常，返回的 set 将依次在 AbstractSet 上实现。此 set 不支持 add() 或 remove() 方法，其迭代器也不支持 remove() 方法。

要实现可修改的映射，编程人员必须另外重写此类的 put 方法（否则将抛出 UnsupportedOperationException），entrySet().iterator() 返回的迭代器也必须另外实现其 remove 方法。

 
**AbstractMap的API**

    abstract Set<Entry<K, V>>     entrySet()
             void                 clear()
             boolean              containsKey(Object key)
             boolean              containsValue(Object value)
             boolean              equals(Object object)
             V                    get(Object key)
             int                  hashCode()
             boolean              isEmpty()
             Set<K>               keySet()
             V                    put(K key, V value)
             void                 putAll(Map<? extends K, ? extends V> map)
             V                    remove(Object key)
             int                  size()
             String               toString()
             Collection<V>        values()
             Object               clone()

 
<a name="anchor4"></a>
# 4. SortedMap

SortedMap的定义如下：

    public interface SortedMap<K,V> extends Map<K,V> { }

SortedMap是一个继承于Map接口的接口。它是一个有序的SortedMap键值映射。  
SortedMap的排序方式有两种：自然排序 或者 用户指定比较器。 插入有序 SortedMap 的所有元素都必须实现 Comparable 接口（或者被指定的比较器所接受）。

另外，所有SortedMap 实现类都应该提供 4 个“标准”构造方法：  
(01) void（无参数）构造方法，它创建一个空的有序映射，按照键的自然顺序进行排序。  
(02) 带有一个 Comparator 类型参数的构造方法，它创建一个空的有序映射，根据指定的比较器进行排序。  
(03) 带有一个 Map 类型参数的构造方法，它创建一个新的有序映射，其键-值映射关系与参数相同，按照键的自然顺序进行排序。  
(04) 带有一个 SortedMap 类型参数的构造方法，它创建一个新的有序映射，其键-值映射关系和排序方法与输入的有序映射相同。无法保证强制实施此建议，因为接口不能包含构造方法。

 

**SortedMap的API**

    // 继承于Map的API
    abstract void                 clear()
    abstract boolean              containsKey(Object key)
    abstract boolean              containsValue(Object value)
    abstract Set<Entry<K, V>>     entrySet()
    abstract boolean              equals(Object object)
    abstract V                    get(Object key)
    abstract int                  hashCode()
    abstract boolean              isEmpty()
    abstract Set<K>               keySet()
    abstract V                    put(K key, V value)
    abstract void                 putAll(Map<? extends K, ? extends V> map)
    abstract V                    remove(Object key)
    abstract int                  size()
    abstract Collection<V>        values()
    // SortedMap新增的API 
    abstract Comparator<? super K>     comparator()
    abstract K                         firstKey()
    abstract SortedMap<K, V>           headMap(K endKey)
    abstract K                         lastKey()
    abstract SortedMap<K, V>           subMap(K startKey, K endKey)
    abstract SortedMap<K, V>           tailMap(K startKey)

 
<a name="anchor5"></a>
# 5 NavigableMap

NavigableMap的定义如下：

    public interface NavigableMap<K,V> extends SortedMap<K,V> { }

NavigableMap是继承于SortedMap的接口。它是一个可导航的键-值对集合，具有了为给定搜索目标报告最接近匹配项的导航方法。  
NavigableMap分别提供了获取“键”、“键-值对”、“键集”、“键-值对集”的相关方法。

 

**NavigableMap的API**

    abstract Entry<K, V>             ceilingEntry(K key)
    abstract Entry<K, V>             firstEntry()
    abstract Entry<K, V>             floorEntry(K key)
    abstract Entry<K, V>             higherEntry(K key)
    abstract Entry<K, V>             lastEntry()
    abstract Entry<K, V>             lowerEntry(K key)
    abstract Entry<K, V>             pollFirstEntry()
    abstract Entry<K, V>             pollLastEntry()
    abstract K                       ceilingKey(K key)
    abstract K                       floorKey(K key)
    abstract K                       higherKey(K key)
    abstract K                       lowerKey(K key)
    abstract NavigableSet<K>         descendingKeySet()
    abstract NavigableSet<K>         navigableKeySet()
    abstract NavigableMap<K, V>      descendingMap()
    abstract NavigableMap<K, V>      headMap(K toKey, boolean inclusive)
    abstract SortedMap<K, V>         headMap(K toKey)
    abstract SortedMap<K, V>         subMap(K fromKey, K toKey)
    abstract NavigableMap<K, V>      subMap(K fromKey, boolean fromInclusive, K toKey, boolean toInclusive)
    abstract SortedMap<K, V>         tailMap(K fromKey)
    abstract NavigableMap<K, V>      tailMap(K fromKey, boolean inclusive)

说明：  
NavigableMap除了继承SortedMap的特性外，它的提供的功能可以分为4类：  
第1类，提供操作键-值对的方法。  
&nbsp;&nbsp;lowerEntry、floorEntry、ceilingEntry 和 higherEntry 方法，它们分别返回与小于、小于等于、大于等于、大于给定键的键关联的 Map.Entry 对象。  
&nbsp;&nbsp;firstEntry、pollFirstEntry、lastEntry 和 pollLastEntry 方法，它们返回和/或移除最小和最大的映射关系（如果存在），否则返回 null。  
第2类，提供操作键的方法。这个和第1类比较类似  
&nbsp;&nbsp;lowerKey、floorKey、ceilingKey 和 higherKey 方法，它们分别返回与小于、小于等于、大于等于、大于给定键的键。  
第3类，获取键集。  
&nbsp;&nbsp;navigableKeySet、descendingKeySet分别获取正序/反序的键集。  
第4类，获取键-值对的子集。

 

<a name="anchor6"></a>
# 6. Dictionary

Dictionary的定义如下：

    public abstract class Dictionary<K,V> {}

NavigableMap是JDK 1.0定义的键值对的接口，它也包括了操作键值对的基本函数。


**Dictionary的API**

    abstract Enumeration<V>     elements()
    abstract V                  get(Object key)
    abstract boolean            isEmpty()
    abstract Enumeration<K>     keys()
    abstract V                  put(K key, V value)
    abstract V                  remove(Object key)
    abstract int                size()

 

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
