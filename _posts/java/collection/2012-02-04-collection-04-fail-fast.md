---
layout: post
title: "Java 集合系列04之 fail-fast总结(通过ArrayList来说明fail-fast的原理、解决办法)"
description: "java collection"
category: java
tags: [java]
date: 2012-02-04 09:01
---

> 前面，我们已经学习了ArrayList。接下来，我们以ArrayList为例，对Iterator的fail-fast机制进行了解。内容包括：：

> **目录**  
> [1 fail-fast简介](#anchor1)   
> [2 fail-fast示例](#anchor2)   
> [3 fail-fast解决办法](#anchor3)   
> [4 fail-fast原理](#anchor4)   
> [5 解决fail-fast的原理](#anchor5) 


  
<a name="anchor1"></a>
# 1. fail-fast简介

fail-fast 机制是java集合(Collection)中的一种错误机制。当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件。  
例如：当某一个线程A通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程A访问集合时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。

在详细介绍fail-fast机制的原理之前，先通过一个示例来认识fail-fast。

  
<a name="anchor2"></a>
# 2 fail-fast示例

示例代码：(FastFailTest.java)

    import java.util.*;
    import java.util.concurrent.*;

    /*
     * @desc java集合中Fast-Fail的测试程序。
     *
     *   fast-fail事件产生的条件：当多个线程对Collection进行操作时，若其中某一个线程通过iterator去遍历集合时，该集合的内容被其他线程所改变；则会抛出ConcurrentModificationException异常。
     *   fast-fail解决办法：通过util.concurrent集合包下的相应类去处理，则不会产生fast-fail事件。
     *
     *   本例中，分别测试ArrayList和CopyOnWriteArrayList这两种情况。ArrayList会产生fast-fail事件，而CopyOnWriteArrayList不会产生fast-fail事件。
     *   (01) 使用ArrayList时，会产生fast-fail事件，抛出ConcurrentModificationException异常；定义如下：
     *            private static List<String> list = new ArrayList<String>();
     *   (02) 使用时CopyOnWriteArrayList，不会产生fast-fail事件；定义如下：
     *            private static List<String> list = new CopyOnWriteArrayList<String>();
     *
     * @author skywang
     */
    public class FastFailTest {

        private static List<String> list = new ArrayList<String>();
        //private static List<String> list = new CopyOnWriteArrayList<String>();
        public static void main(String[] args) {
        
            // 同时启动两个线程对list进行操作！
            new ThreadOne().start();
            new ThreadTwo().start();
        }

        private static void printAll() {
            System.out.println("");

            String value = null;
            Iterator iter = list.iterator();
            while(iter.hasNext()) {
                value = (String)iter.next();
                System.out.print(value+", ");
            }
        }

        /**
         * 向list中依次添加0,1,2,3,4,5，每添加一个数之后，就通过printAll()遍历整个list
         */
        private static class ThreadOne extends Thread {
            public void run() {
                int i = 0;
                while (i<6) {
                    list.add(String.valueOf(i));
                    printAll();
                    i++;
                }
            }
        }

        /**
         * 向list中依次添加10,11,12,13,14,15，每添加一个数之后，就通过printAll()遍历整个list
         */
        private static class ThreadTwo extends Thread {
            public void run() {
                int i = 10;
                while (i<16) {
                    list.add(String.valueOf(i));
                    printAll();
                    i++;
                }
            }
        }

    }

**运行结果**

运行该代码，抛出异常java.util.ConcurrentModificationException！即，产生fail-fast事件！

**结果说明**

(01) FastFailTest中通过 new ThreadOne().start() 和 new ThreadTwo().start() 同时启动两个线程去操作list。  
>    ThreadOne线程：向list中依次添加0,1,2,3,4,5。每添加一个数之后，就通过printAll()遍历整个list。  
>    ThreadTwo线程：向list中依次添加10,11,12,13,14,15。每添加一个数之后，就通过printAll()遍历整个list。

(02) 当某一个线程遍历list的过程中，list的内容被另外一个线程所改变了；就会抛出ConcurrentModificationException异常，产生fail-fast事件。

 
<a name="anchor3"></a>
# 3. fail-fast解决办法

fail-fast机制，是一种错误检测机制。它只能被用来检测错误，因为JDK并不保证fail-fast机制一定会发生。若在多线程环境下使用fail-fast机制的集合，建议使用“java.util.concurrent包下的类”去取代“java.util包下的类”。  
所以，本例中只需要将ArrayList替换成java.util.concurrent包下对应的类即可。
即，将代码

    private static List<String> list = new ArrayList<String>();

替换为

    private static List<String> list = new CopyOnWriteArrayList<String>();

则可以解决该办法。

 
<a name="anchor4"></a>
# 4. fail-fast原理

产生fail-fast事件，是通过抛出ConcurrentModificationException异常来触发的。  
那么，ArrayList是如何抛出ConcurrentModificationException异常的呢?

我们知道，ConcurrentModificationException是在操作Iterator时抛出的异常。我们先看看Iterator的源码。ArrayList的Iterator是在父类AbstractList.java中实现的。代码如下： 

    package java.util;

    public abstract class AbstractList<E> extends AbstractCollection<E> implements List<E> {

        ...

        // AbstractList中唯一的属性
        // 用来记录List修改的次数：每修改一次(添加/删除等操作)，将modCount+1
        protected transient int modCount = 0;

        // 返回List对应迭代器。实际上，是返回Itr对象。
        public Iterator<E> iterator() {
            return new Itr();
        }

        // Itr是Iterator(迭代器)的实现类
        private class Itr implements Iterator<E> {
            int cursor = 0;

            int lastRet = -1;

            // 修改数的记录值。
            // 每次新建Itr()对象时，都会保存新建该对象时对应的modCount；
            // 以后每次遍历List中的元素的时候，都会比较expectedModCount和modCount是否相等；
            // 若不相等，则抛出ConcurrentModificationException异常，产生fail-fast事件。
            int expectedModCount = modCount;

            public boolean hasNext() {
                return cursor != size();
            }

            public E next() {
                // 获取下一个元素之前，都会判断“新建Itr对象时保存的modCount”和“当前的modCount”是否相等；
                // 若不相等，则抛出ConcurrentModificationException异常，产生fail-fast事件。
                checkForComodification();
                try {
                    E next = get(cursor);
                    lastRet = cursor++;
                    return next;
                } catch (IndexOutOfBoundsException e) {
                    checkForComodification();
                    throw new NoSuchElementException();
                }
            }

            public void remove() {
                if (lastRet == -1)
                    throw new IllegalStateException();
                checkForComodification();

                try {
                    AbstractList.this.remove(lastRet);
                    if (lastRet < cursor)
                        cursor--;
                    lastRet = -1;
                    expectedModCount = modCount;
                } catch (IndexOutOfBoundsException e) {
                    throw new ConcurrentModificationException();
                }
            }

            final void checkForComodification() {
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
            }
        }

        ...
    }

从中，我们可以发现在调用 next() 和 remove()时，都会执行 checkForComodification()。若 “modCount 不等于 expectedModCount”，则抛出ConcurrentModificationException异常，产生fail-fast事件。

要搞明白 fail-fast机制，我们就要需要理解什么时候“modCount 不等于 expectedModCount”！  
从Itr类中，我们知道 expectedModCount 在创建Itr对象时，被赋值为 modCount。通过Itr，我们知道：expectedModCount不可能被修改为不等于 modCount。所以，需要考证的就是modCount何时会被修改。

接下来，我们查看ArrayList的源码，来看看modCount是如何被修改的。

    package java.util;

    public class ArrayList<E> extends AbstractList<E>
            implements List<E>, RandomAccess, Cloneable, java.io.Serializable
    {

        ...

        // list中容量变化时，对应的同步函数
        public void ensureCapacity(int minCapacity) {
            modCount++;
            int oldCapacity = elementData.length;
            if (minCapacity > oldCapacity) {
                Object oldData[] = elementData;
                int newCapacity = (oldCapacity * 3)/2 + 1;
                if (newCapacity < minCapacity)
                    newCapacity = minCapacity;
                // minCapacity is usually close to size, so this is a win:
                elementData = Arrays.copyOf(elementData, newCapacity);
            }
        }


        // 添加元素到队列最后
        public boolean add(E e) {
            // 修改modCount
            ensureCapacity(size + 1);  // Increments modCount!!
            elementData[size++] = e;
            return true;
        }


        // 添加元素到指定的位置
        public void add(int index, E element) {
            if (index > size || index < 0)
                throw new IndexOutOfBoundsException(
                "Index: "+index+", Size: "+size);

            // 修改modCount
            ensureCapacity(size+1);  // Increments modCount!!
            System.arraycopy(elementData, index, elementData, index + 1,
                 size - index);
            elementData[index] = element;
            size++;
        }

        // 添加集合
        public boolean addAll(Collection<? extends E> c) {
            Object[] a = c.toArray();
            int numNew = a.length;
            // 修改modCount
            ensureCapacity(size + numNew);  // Increments modCount
            System.arraycopy(a, 0, elementData, size, numNew);
            size += numNew;
            return numNew != 0;
        }
       

        // 删除指定位置的元素 
        public E remove(int index) {
            RangeCheck(index);

            // 修改modCount
            modCount++;
            E oldValue = (E) elementData[index];

            int numMoved = size - index - 1;
            if (numMoved > 0)
                System.arraycopy(elementData, index+1, elementData, index, numMoved);
            elementData[--size] = null; // Let gc do its work

            return oldValue;
        }


        // 快速删除指定位置的元素 
        private void fastRemove(int index) {

            // 修改modCount
            modCount++;
            int numMoved = size - index - 1;
            if (numMoved > 0)
                System.arraycopy(elementData, index+1, elementData, index,
                                 numMoved);
            elementData[--size] = null; // Let gc do its work
        }

        // 清空集合
        public void clear() {
            // 修改modCount
            modCount++;

            // Let gc do its work
            for (int i = 0; i < size; i++)
                elementData[i] = null;

            size = 0;
        }

        ...
    }

从中，我们发现：无论是add()、remove()，还是clear()，只要涉及到修改集合中的元素个数时，都会改变modCount的值。

接下来，我们再系统的梳理一下fail-fast是怎么产生的。步骤如下：  
(01) 新建了一个ArrayList，名称为arrayList。  
(02) 向arrayList中添加内容。  
(03) 新建一个“线程a”，并在“线程a”中通过Iterator反复的读取arrayList的值。  
(04) 新建一个“线程b”，在“线程b”中删除arrayList中的一个“节点A”。  
(05) 这时，就会产生有趣的事件了。  
> 在某一时刻，“线程a”创建了arrayList的Iterator。此时“节点A”仍然存在于arrayList中，创建arrayList时，expectedModCount = modCount(假设它们此时的值为N)。  
> 在“线程a”在遍历arrayList过程中的某一时刻，“线程b”执行了，并且“线程b”删除了arrayList中的“节点A”。“线程b”执行remove()进行删除操作时，在remove()中执行了“modCount++”，此时modCount变成了N+1！  
> “线程a”接着遍历，当它执行到next()函数时，调用checkForComodification()比较“expectedModCount”和“modCount”的大小；而“expectedModCount=N”，“modCount=N+1”,这样，便抛出ConcurrentModificationException异常，产生fail-fast事件。

至此，我们就完全了解了fail-fast是如何产生的！  
即，当多个线程对同一个集合进行操作的时候，某线程访问集合的过程中，该集合的内容被其他线程所改变(即其它线程通过add、remove、clear等方法，改变了modCount的值)；这时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。

 
<a name="anchor5"></a>
# 5. 解决fail-fast的原理

上面，说明了“解决fail-fast机制的办法”，也知道了“fail-fast产生的根本原因”。接下来，我们再进一步谈谈java.util.concurrent包中是如何解决fail-fast事件的。  
还是以和ArrayList对应的CopyOnWriteArrayList进行说明。我们先看看CopyOnWriteArrayList的源码：

    package java.util.concurrent;
    import java.util.*;
    import java.util.concurrent.locks.*;
    import sun.misc.Unsafe;

    public class CopyOnWriteArrayList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

        ...

        // 返回集合对应的迭代器
        public Iterator<E> iterator() {
            return new COWIterator<E>(getArray(), 0);
        }

        ...
       
        private static class COWIterator<E> implements ListIterator<E> {
            private final Object[] snapshot;

            private int cursor;

            private COWIterator(Object[] elements, int initialCursor) {
                cursor = initialCursor;
                // 新建COWIterator时，将集合中的元素保存到一个新的拷贝数组中。
                // 这样，当原始集合的数据改变，拷贝数据中的值也不会变化。
                snapshot = elements;
            }

            public boolean hasNext() {
                return cursor < snapshot.length;
            }

            public boolean hasPrevious() {
                return cursor > 0;
            }

            public E next() {
                if (! hasNext())
                    throw new NoSuchElementException();
                return (E) snapshot[cursor++];
            }

            public E previous() {
                if (! hasPrevious())
                    throw new NoSuchElementException();
                return (E) snapshot[--cursor];
            }

            public int nextIndex() {
                return cursor;
            }

            public int previousIndex() {
                return cursor-1;
            }

            public void remove() {
                throw new UnsupportedOperationException();
            }

            public void set(E e) {
                throw new UnsupportedOperationException();
            }

            public void add(E e) {
                throw new UnsupportedOperationException();
            }
        }
      
        ...

    }

从中，我们可以看出:

(01) 和ArrayList继承于AbstractList不同，CopyOnWriteArrayList没有继承于AbstractList，它仅仅只是实现了List接口。  
(02) ArrayList的iterator()函数返回的Iterator是在AbstractList中实现的；而CopyOnWriteArrayList是自己实现Iterator。  
(03) ArrayList的Iterator实现类中调用next()时，会“调用checkForComodification()比较‘expectedModCount’和‘modCount’的大小”；但是，CopyOnWriteArrayList的Iterator实现类中，没有所谓的checkForComodification()，更不会抛出ConcurrentModificationException异常！ 

 

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
