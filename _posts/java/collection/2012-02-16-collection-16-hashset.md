---
layout: post
title: "Java 集合系列16之 HashSet详细介绍(源码解析)和使用示例"
description: "java collection"
category: java
tags: [java]
date: 2012-02-16 09:01
---

 
> 这一章，我们对HashSet进行学习。  
> 我们先对HashSet有个整体认识，然后再学习它的源码，最后再通过实例来学会使用HashSet。

> **目录**  
> [第1部分 HashSet介绍](#anchor1)   
> [第2部分 HashSet数据结构](#anchor2)   
> [第3部分 HashSet源码解析(基于JDK1.6.0_45)](#anchor3)   
> [第4部分 HashSet遍历方式](#anchor4)   
> [第5部分 HashSet示例](#anchor5)   

 
<a name="anchor1"></a>
# 第1部分 HashSet介绍

## HashSet 简介

HashSet 是一个没有重复元素的集合。  
它是由HashMap实现的，不保证元素的顺序，而且HashSet允许使用 null 元素。  
HashSet是非同步的。如果多个线程同时访问一个哈希 set，而其中至少一个线程修改了该 set，那么它必须 保持外部同步。这通常是通过对自然封装该 set 的对象执行同步操作来完成的。如果不存在这样的对象，则应该使用 Collections.synchronizedSet 方法来“包装” set。最好在创建时完成这一操作，以防止对该 set 进行意外的不同步访问：

    Set s = Collections.synchronizedSet(new HashSet(...));

HashSet通过iterator()返回的迭代器是fail-fast的。

 

## HashSet的构造函数

    // 默认构造函数
    public HashSet() 

    // 带集合的构造函数
    public HashSet(Collection<? extends E> c) 

    // 指定HashSet初始容量和加载因子的构造函数
    public HashSet(int initialCapacity, float loadFactor) 

    // 指定HashSet初始容量的构造函数
    public HashSet(int initialCapacity) 

    // 指定HashSet初始容量和加载因子的构造函数,dummy没有任何作用
    HashSet(int initialCapacity, float loadFactor, boolean dummy) 

## HashSet的主要API

    boolean         add(E object)
    void            clear()
    Object          clone()
    boolean         contains(Object object)
    boolean         isEmpty()
    Iterator<E>     iterator()
    boolean         remove(Object object)
    int             size()

 
 
<a name="anchor2"></a>
# 第2部分 HashSet数据结构

HashSet的继承关系如下：

    java.lang.Object
       ↳     java.util.AbstractCollection<E>
             ↳     java.util.AbstractSet<E>
                   ↳     java.util.HashSet<E>

public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable { }

 

HashSet与Map关系如下图：

![img](/media/pic/java/collection/collection16.jpg)

从图中可以看出：  
(01) HashSet继承于AbstractSet，并且实现了Set接口。  
(02) HashSet的本质是一个"没有重复元素"的集合，它是通过HashMap实现的。HashSet中含有一个"HashMap类型的成员变量"map，HashSet的操作函数，实际上都是通过map实现的。

 
 
<a name="anchor3"></a>
# 第3部分 HashSet源码解析(基于JDK1.6.0_45)

为了更了解HashSet的原理，下面对HashSet源码代码作出分析。

    package java.util;

    public class HashSet<E>
        extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable
    {
        static final long serialVersionUID = -5024744406713321676L;

        // HashSet是通过map(HashMap对象)保存内容的
        private transient HashMap<E,Object> map;

        // PRESENT是向map中插入key-value对应的value
        // 因为HashSet中只需要用到key，而HashMap是key-value键值对；
        // 所以，向map中添加键值对时，键值对的值固定是PRESENT
        private static final Object PRESENT = new Object();

        // 默认构造函数
        public HashSet() {
            // 调用HashMap的默认构造函数，创建map
            map = new HashMap<E,Object>();
        }

        // 带集合的构造函数
        public HashSet(Collection<? extends E> c) {
            // 创建map。
            // 为什么要调用Math.max((int) (c.size()/.75f) + 1, 16)，从 (c.size()/.75f) + 1 和 16 中选择一个比较大的树呢？        
            // 首先，说明(c.size()/.75f) + 1
            //   因为从HashMap的效率(时间成本和空间成本)考虑，HashMap的加载因子是0.75。
            //   当HashMap的“阈值”(阈值=HashMap总的大小*加载因子) < “HashMap实际大小”时，
            //   就需要将HashMap的容量翻倍。
            //   所以，(c.size()/.75f) + 1 计算出来的正好是总的空间大小。
            // 接下来，说明为什么是 16 。
            //   HashMap的总的大小，必须是2的指数倍。若创建HashMap时，指定的大小不是2的指数倍；
            //   HashMap的构造函数中也会重新计算，找出比“指定大小”大的最小的2的指数倍的数。
            //   所以，这里指定为16是从性能考虑。避免重复计算。
            map = new HashMap<E,Object>(Math.max((int) (c.size()/.75f) + 1, 16));
            // 将集合(c)中的全部元素添加到HashSet中
            addAll(c);
        }

        // 指定HashSet初始容量和加载因子的构造函数
        public HashSet(int initialCapacity, float loadFactor) {
            map = new HashMap<E,Object>(initialCapacity, loadFactor);
        }

        // 指定HashSet初始容量的构造函数
        public HashSet(int initialCapacity) {
            map = new HashMap<E,Object>(initialCapacity);
        }

        HashSet(int initialCapacity, float loadFactor, boolean dummy) {
            map = new LinkedHashMap<E,Object>(initialCapacity, loadFactor);
        }

        // 返回HashSet的迭代器
        public Iterator<E> iterator() {
            // 实际上返回的是HashMap的“key集合的迭代器”
            return map.keySet().iterator();
        }

        public int size() {
            return map.size();
        }

        public boolean isEmpty() {
            return map.isEmpty();
        }

        public boolean contains(Object o) {
            return map.containsKey(o);
        }

        // 将元素(e)添加到HashSet中
        public boolean add(E e) {
            return map.put(e, PRESENT)==null;
        }

        // 删除HashSet中的元素(o)
        public boolean remove(Object o) {
            return map.remove(o)==PRESENT;
        }

        public void clear() {
            map.clear();
        }

        // 克隆一个HashSet，并返回Object对象
        public Object clone() {
            try {
                HashSet<E> newSet = (HashSet<E>) super.clone();
                newSet.map = (HashMap<E, Object>) map.clone();
                return newSet;
            } catch (CloneNotSupportedException e) {
                throw new InternalError();
            }
        }

        // java.io.Serializable的写入函数
        // 将HashSet的“总的容量，加载因子，实际容量，所有的元素”都写入到输出流中
        private void writeObject(java.io.ObjectOutputStream s)
            throws java.io.IOException {
            // Write out any hidden serialization magic
            s.defaultWriteObject();

            // Write out HashMap capacity and load factor
            s.writeInt(map.capacity());
            s.writeFloat(map.loadFactor());

            // Write out size
            s.writeInt(map.size());

            // Write out all elements in the proper order.
            for (Iterator i=map.keySet().iterator(); i.hasNext(); )
                s.writeObject(i.next());
        }


        // java.io.Serializable的读取函数
        // 将HashSet的“总的容量，加载因子，实际容量，所有的元素”依次读出
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            // Read in any hidden serialization magic
            s.defaultReadObject();

            // Read in HashMap capacity and load factor and create backing HashMap
            int capacity = s.readInt();
            float loadFactor = s.readFloat();
            map = (((HashSet)this) instanceof LinkedHashSet ?
                   new LinkedHashMap<E,Object>(capacity, loadFactor) :
                   new HashMap<E,Object>(capacity, loadFactor));

            // Read in size
            int size = s.readInt();

            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                E e = (E) s.readObject();
                map.put(e, PRESENT);
            }
        }
    }

说明： HashSet的代码实际上非常简单，通过上面的注释应该很能够看懂。它是通过HashMap实现的，若对HashSet的理解有困难，建议先学习以下HashMap；学完HashMap之后，在学习HashSet就非常容易了。

 
 
<a name="anchor4"></a>
# 第4部分 HashSet遍历方式

## 4.1 通过Iterator遍历HashSet

第一步：根据iterator()获取HashSet的迭代器。

第二步：遍历迭代器获取各个元素。

    // 假设set是HashSet对象
    for(Iterator iterator = set.iterator();
           iterator.hasNext(); ) { 
        iterator.next();
    }   

 

## 4.2 通过for-each遍历HashSet

第一步：根据toArray()获取HashSet的元素集合对应的数组。

第二步：遍历数组，获取各个元素。

    // 假设set是HashSet对象，并且set中元素是String类型
    String[] arr = (String[])set.toArray(new String[0]);
    for (String str:arr)
        System.out.printf("for each : %s\n", str);

HashSet的遍历测试程序如下： 

    import java.util.Random;
    import java.util.Iterator;
    import java.util.HashSet;

    /*
     * @desc 介绍HashSet遍历方法
     *
     * @author skywang
     */
    public class HashSetIteratorTest {

        public static void main(String[] args) {
            // 新建HashSet
            HashSet set = new HashSet();

            // 添加元素 到HashSet中
            for (int i=0; i<5; i++)
                set.add(""+i);

            // 通过Iterator遍历HashSet
            iteratorHashSet(set) ;

            // 通过for-each遍历HashSet
            foreachHashSet(set);
        }

        /*
         * 通过Iterator遍历HashSet。推荐方式
         */
        private static void iteratorHashSet(HashSet set) {
            for(Iterator iterator = set.iterator();
                   iterator.hasNext(); ) {
                System.out.printf("iterator : %s\n", iterator.next());
            }
        }

        /*
         * 通过for-each遍历HashSet。不推荐！此方法需要先将Set转换为数组
         */
        private static void foreachHashSet(HashSet set) {
            String[] arr = (String[])set.toArray(new String[0]);
            for (String str:arr)
                System.out.printf("for each : %s\n", str);
        }
    }

运行结果： 

iterator : 3
iterator : 2
iterator : 1
iterator : 0
iterator : 4
for each : 3
for each : 2
for each : 1
for each : 0
for each : 4

 
 
<a name="anchor5"></a>
# 第5部分 HashSet示例

下面我们通过实例学习如何使用HashSet

import java.util.Iterator;
import java.util.HashSet;

/*
 * @desc HashSet常用API的使用。
 *
 * @author skywang
 */
public class HashSetTest {

    public static void main(String[] args) {
        // HashSet常用API
        testHashSetAPIs() ;
    }

    /*
     * HashSet除了iterator()和add()之外的其它常用API
     */
    private static void testHashSetAPIs() {
        // 新建HashSet
        HashSet set = new HashSet();

        // 将元素添加到Set中
        set.add("a");
        set.add("b");
        set.add("c");
        set.add("d");
        set.add("e");

        // 打印HashSet的实际大小
        System.out.printf("size : %d\n", set.size());

        // 判断HashSet是否包含某个值
        System.out.printf("HashSet contains a :%s\n", set.contains("a"));
        System.out.printf("HashSet contains g :%s\n", set.contains("g"));

        // 删除HashSet中的“e”
        set.remove("e");

        // 将Set转换为数组
        String[] arr = (String[])set.toArray(new String[0]);
        for (String str:arr)
            System.out.printf("for each : %s\n", str);

        // 新建一个包含b、c、f的HashSet
        HashSet otherset = new HashSet();
        otherset.add("b");
        otherset.add("c");
        otherset.add("f");

        // 克隆一个removeset，内容和set一模一样
        HashSet removeset = (HashSet)set.clone();
        // 删除“removeset中，属于otherSet的元素”
        removeset.removeAll(otherset);
        // 打印removeset
        System.out.printf("removeset : %s\n", removeset);

        // 克隆一个retainset，内容和set一模一样
        HashSet retainset = (HashSet)set.clone();
        // 保留“retainset中，属于otherSet的元素”
        retainset.retainAll(otherset);
        // 打印retainset
        System.out.printf("retainset : %s\n", retainset);


        // 遍历HashSet
        for(Iterator iterator = set.iterator();
               iterator.hasNext(); ) 
            System.out.printf("iterator : %s\n", iterator.next());

        // 清空HashSet
        set.clear();

        // 输出HashSet是否为空
        System.out.printf("%s\n", set.isEmpty()?"set is empty":"set is not empty");
    }

}

运行结果： 

    size : 5
    HashSet contains a :true
    HashSet contains g :false
    for each : d
    for each : b
    for each : c
    for each : a
    removeset : [d, a]
    retainset : [b, c]
    iterator : d
    iterator : b
    iterator : c
    iterator : a
    set is empty

     

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
