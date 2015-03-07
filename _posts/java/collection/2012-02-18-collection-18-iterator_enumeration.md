---
layout: post
title: "Java 集合系列18之 Iterator和Enumeration比较"
description: "java collection"
category: java
tags: [java]
date: 2012-02-18 09:01
---

 
> 这一章，我们对Iterator和Enumeration进行比较学习。

> **目录**  
> [第1部分 Iterator和Enumeration区别](#anchor1)   
> [第2部分 Iterator和Enumeration实例](#anchor2)   


 
 
<a name="anchor1"></a>
# 第1部分 Iterator和Enumeration区别

在Java集合中，我们通常都通过 “Iterator(迭代器)” 或 “Enumeration(枚举类)” 去遍历集合。今天，我们就一起学习一下它们之间到底有什么区别。

我们先看看 Enumeration.java 和 Iterator.java的源码，再说它们的区别。

Enumeration是一个接口，它的源码如下：

    package java.util;

    public interface Enumeration<E> {

        boolean hasMoreElements();

        E nextElement();
    }

Iterator也是一个接口，它的源码如下：

    package java.util;

    public interface Iterator<E> {
        boolean hasNext();

        E next();

        void remove();
    }

看完代码了，我们再来说说它们之间的区别。

**(01) 函数接口不同**  
&nbsp;&nbsp;&nbsp;&nbsp; Enumeration只有2个函数接口。通过Enumeration，我们只能读取集合的数据，而不能对数据进行修改。  
&nbsp;&nbsp;&nbsp;&nbsp;Iterator只有3个函数接口。Iterator除了能读取集合的数据之外，也能数据进行删除操作。

**(02) Iterator支持fail-fast机制，而Enumeration不支持。**  
&nbsp;&nbsp;&nbsp;&nbsp;Enumeration 是JDK 1.0添加的接口。使用到它的函数包括Vector、Hashtable等类，这些类都是JDK 1.0中加入的，Enumeration存在的目的就是为它们提供遍历接口。Enumeration本身并没有支持同步，而在Vector、Hashtable实现Enumeration时，添加了同步。  
&nbsp;&nbsp;&nbsp;&nbsp;而Iterator 是JDK 1.2才添加的接口，它也是为了HashMap、ArrayList等集合提供遍历接口。Iterator是支持fail-fast机制的：当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件。

 
 
<a name="anchor2"></a>
# 第2部分 Iterator和Enumeration实例

下面，我们编写一个Hashtable，然后分别通过 Iterator 和 Enumeration 去遍历它，比较它们的效率。代码如下：

    import java.util.Enumeration;
    import java.util.Hashtable;
    import java.util.Iterator;
    import java.util.Map.Entry;
    import java.util.Random;

    /*
     * 测试分别通过 Iterator 和 Enumeration 去遍历Hashtable
     * @author skywang
     */
    public class IteratorEnumeration {

        public static void main(String[] args) {
            int val;
            Random r = new Random();
            Hashtable table = new Hashtable();
            for (int i=0; i<100000; i++) {
                // 随机获取一个[0,100)之间的数字
                val = r.nextInt(100);
                table.put(String.valueOf(i), val);
            }

            // 通过Iterator遍历Hashtable
            iterateHashtable(table) ;

            // 通过Enumeration遍历Hashtable
            enumHashtable(table);
        }
        
        /*
         * 通过Iterator遍历Hashtable
         */
        private static void iterateHashtable(Hashtable table) {
            long startTime = System.currentTimeMillis();

            Iterator iter = table.entrySet().iterator();
            while(iter.hasNext()) {
                //System.out.println("iter:"+iter.next());
                iter.next();
            }

            long endTime = System.currentTimeMillis();
            countTime(startTime, endTime);
        }
        
        /*
         * 通过Enumeration遍历Hashtable
         */
        private static void enumHashtable(Hashtable table) {
            long startTime = System.currentTimeMillis();

            Enumeration enu = table.elements();
            while(enu.hasMoreElements()) {
                //System.out.println("enu:"+enu.nextElement());
                enu.nextElement();
            }

            long endTime = System.currentTimeMillis();
            countTime(startTime, endTime);
        }

        private static void countTime(long start, long end) {
            System.out.println("time: "+(end-start)+"ms");
        }
    }

运行结果如下：

    time: 9ms
    time: 5ms

从中，我们可以看出。Enumeration 比 Iterator 的遍历速度更快。为什么呢？
这是因为，Hashtable中Iterator是通过Enumeration去实现的，而且Iterator添加了对fail-fast机制的支持；所以，执行的操作自然要多一些。

 

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
