---
layout: post
title: "Java 集合系列10之 HashMap详细介绍(源码解析)和使用示例"
description: "java collection"
category: java
tags: [java]
date: 2012-02-10 09:01
---

 

> 这一章，我们对HashMap进行学习。  
> 我们先对HashMap有个整体认识，然后再学习它的源码，最后再通过实例来学会使用HashMap。内容包括：  

> **目录**  
> [第1部分 HashMap介绍](#anchor1)   
> [第2部分 HashMap数据结构](#anchor2)   
> [第3部分 HashMap源码解析(基于JDK1.6.0_45)](#anchor3)   
> &nbsp;&nbsp;[第3.1部分 HashMap的“拉链法”相关内容](#anchor3_1)   
> &nbsp;&nbsp;[第3.2部分 HashMap的构造函数](#anchor3_2)   
> &nbsp;&nbsp;[第3.3部分 HashMap的主要对外接口](#anchor3_3)   
> &nbsp;&nbsp;[第3.4部分 HashMap实现的Cloneable接口](#anchor3_4)   
> &nbsp;&nbsp;[第3.5部分 HashMap实现的Serializable接口](#anchor3_5)   
> [第4部分 HashMap遍历方式](#anchor4)   
> [第5部分 HashMap示例](#anchor5)   


 
 
<a name="anchor1"></a>
# 第1部分 HashMap介绍

## HashMap简介

HashMap 是一个散列表，它存储的内容是键值对(key-value)映射。  
HashMap 继承于AbstractMap，实现了Map、Cloneable、java.io.Serializable接口。  
HashMap 的实现不是同步的，这意味着它不是线程安全的。它的key、value都可以为null。此外，HashMap中的映射不是有序的。

HashMap 的实例有两个参数影响其性能：“初始容量” 和 “加载因子”。容量 是哈希表中桶的数量，初始容量 只是哈希表在创建时的容量。加载因子 是哈希表在其容量自动增加之前可以达到多满的一种尺度。当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 rehash 操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数。  
通常，默认加载因子是 0.75, 这是在时间和空间成本上寻求一种折衷。加载因子过高虽然减少了空间开销，但同时也增加了查询成本（在大多数 HashMap 类的操作中，包括 get 和 put 操作，都反映了这一点）。在设置初始容量时应该考虑到映射中所需的条目数及其加载因子，以便最大限度地减少 rehash 操作次数。如果初始容量大于最大条目数除以加载因子，则不会发生 rehash 操作。

 

## HashMap的构造函数

HashMap共有4个构造函数,如下：

    // 默认构造函数。
    HashMap()

    // 指定“容量大小”的构造函数
    HashMap(int capacity)

    // 指定“容量大小”和“加载因子”的构造函数
    HashMap(int capacity, float loadFactor)

    // 包含“子Map”的构造函数
    HashMap(Map<? extends K, ? extends V> map)

 

## HashMap的API

    void                 clear()
    Object               clone()
    boolean              containsKey(Object key)
    boolean              containsValue(Object value)
    Set<Entry<K, V>>     entrySet()
    V                    get(Object key)
    boolean              isEmpty()
    Set<K>               keySet()
    V                    put(K key, V value)
    void                 putAll(Map<? extends K, ? extends V> map)
    V                    remove(Object key)
    int                  size()
    Collection<V>        values()

 
 
<a name="anchor2"></a>
# 第2部分 HashMap数据结构

HashMap的继承关系

    java.lang.Object
       ↳     java.util.AbstractMap<K, V>
             ↳     java.util.HashMap<K, V>

HashMap的声明

    public class HashMap<K,V>
        extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable { }

 

HashMap与Map关系如下图：

![img](/media/pic/java/collection/collection10.jpg)

从图中可以看出：  
(01) HashMap继承于AbstractMap类，实现了Map接口。Map是"key-value键值对"接口，AbstractMap实现了"键值对"的通用函数接口。  
(02) HashMap是通过"拉链法"实现的哈希表。它包括几个重要的成员变量：table, size, threshold, loadFactor, modCount。  
> table是一个Entry[]数组类型，而Entry实际上就是一个单向链表。哈希表的"key-value键值对"都是存储在Entry数组中的。  
> size是HashMap的大小，它是HashMap保存的键值对的数量。  
> threshold是HashMap的阈值，用于判断是否需要调整HashMap的容量。threshold的值="容量*加载因子"，当HashMap中存储数据的数量达到threshold时，就需要将HashMap的容量加倍。  
> loadFactor就是加载因子。  
> modCount是用来实现fail-fast机制的。

 
 
<a name="anchor3"></a>
# 第3部分 HashMap源码解析(基于JDK1.6.0_45)

为了更了解HashMap的原理，下面对HashMap源码代码作出分析。  
在阅读源码时，建议参考后面的说明来建立对HashMap的整体认识，这样更容易理解HashMap。

    package java.util;
    import java.io.*;

    public class HashMap<K,V>
        extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable
    {

        // 默认的初始容量是16，必须是2的幂。
        static final int DEFAULT_INITIAL_CAPACITY = 16;

        // 最大容量（必须是2的幂且小于2的30次方，传入容量过大将被这个值替换）
        static final int MAXIMUM_CAPACITY = 1 << 30;

        // 默认加载因子
        static final float DEFAULT_LOAD_FACTOR = 0.75f;

        // 存储数据的Entry数组，长度是2的幂。
        // HashMap是采用拉链法实现的，每一个Entry本质上是一个单向链表
        transient Entry[] table;

        // HashMap的大小，它是HashMap保存的键值对的数量
        transient int size;

        // HashMap的阈值，用于判断是否需要调整HashMap的容量（threshold = 容量*加载因子）
        int threshold;

        // 加载因子实际大小
        final float loadFactor;

        // HashMap被改变的次数
        transient volatile int modCount;

        // 指定“容量大小”和“加载因子”的构造函数
        public HashMap(int initialCapacity, float loadFactor) {
            if (initialCapacity < 0)
                throw new IllegalArgumentException("Illegal initial capacity: " +
                                                   initialCapacity);
            // HashMap的最大容量只能是MAXIMUM_CAPACITY
            if (initialCapacity > MAXIMUM_CAPACITY)
                initialCapacity = MAXIMUM_CAPACITY;
            if (loadFactor <= 0 || Float.isNaN(loadFactor))
                throw new IllegalArgumentException("Illegal load factor: " +
                                                   loadFactor);

            // 找出“大于initialCapacity”的最小的2的幂
            int capacity = 1;
            while (capacity < initialCapacity)
                capacity <<= 1;

            // 设置“加载因子”
            this.loadFactor = loadFactor;
            // 设置“HashMap阈值”，当HashMap中存储数据的数量达到threshold时，就需要将HashMap的容量加倍。
            threshold = (int)(capacity * loadFactor);
            // 创建Entry数组，用来保存数据
            table = new Entry[capacity];
            init();
        }


        // 指定“容量大小”的构造函数
        public HashMap(int initialCapacity) {
            this(initialCapacity, DEFAULT_LOAD_FACTOR);
        }

        // 默认构造函数。
        public HashMap() {
            // 设置“加载因子”
            this.loadFactor = DEFAULT_LOAD_FACTOR;
            // 设置“HashMap阈值”，当HashMap中存储数据的数量达到threshold时，就需要将HashMap的容量加倍。
            threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
            // 创建Entry数组，用来保存数据
            table = new Entry[DEFAULT_INITIAL_CAPACITY];
            init();
        }

        // 包含“子Map”的构造函数
        public HashMap(Map<? extends K, ? extends V> m) {
            this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                          DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
            // 将m中的全部元素逐个添加到HashMap中
            putAllForCreate(m);
        }

        static int hash(int h) {
            h ^= (h >>> 20) ^ (h >>> 12);
            return h ^ (h >>> 7) ^ (h >>> 4);
        }

        // 返回索引值
        // h & (length-1)保证返回值的小于length
        static int indexFor(int h, int length) {
            return h & (length-1);
        }

        public int size() {
            return size;
        }

        public boolean isEmpty() {
            return size == 0;
        }

        // 获取key对应的value
        public V get(Object key) {
            if (key == null)
                return getForNullKey();
            // 获取key的hash值
            int hash = hash(key.hashCode());
            // 在“该hash值对应的链表”上查找“键值等于key”的元素
            for (Entry<K,V> e = table[indexFor(hash, table.length)];
                 e != null;
                 e = e.next) {
                Object k;
                if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                    return e.value;
            }
            return null;
        }

        // 获取“key为null”的元素的值
        // HashMap将“key为null”的元素存储在table[0]位置！
        private V getForNullKey() {
            for (Entry<K,V> e = table[0]; e != null; e = e.next) {
                if (e.key == null)
                    return e.value;
            }
            return null;
        }

        // HashMap是否包含key
        public boolean containsKey(Object key) {
            return getEntry(key) != null;
        }

        // 返回“键为key”的键值对
        final Entry<K,V> getEntry(Object key) {
            // 获取哈希值
            // HashMap将“key为null”的元素存储在table[0]位置，“key不为null”的则调用hash()计算哈希值
            int hash = (key == null) ? 0 : hash(key.hashCode());
            // 在“该hash值对应的链表”上查找“键值等于key”的元素
            for (Entry<K,V> e = table[indexFor(hash, table.length)];
                 e != null;
                 e = e.next) {
                Object k;
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            }
            return null;
        }

        // 将“key-value”添加到HashMap中
        public V put(K key, V value) {
            // 若“key为null”，则将该键值对添加到table[0]中。
            if (key == null)
                return putForNullKey(value);
            // 若“key不为null”，则计算该key的哈希值，然后将其添加到该哈希值对应的链表中。
            int hash = hash(key.hashCode());
            int i = indexFor(hash, table.length);
            for (Entry<K,V> e = table[i]; e != null; e = e.next) {
                Object k;
                // 若“该key”对应的键值对已经存在，则用新的value取代旧的value。然后退出！
                if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                    V oldValue = e.value;
                    e.value = value;
                    e.recordAccess(this);
                    return oldValue;
                }
            }

            // 若“该key”对应的键值对不存在，则将“key-value”添加到table中
            modCount++;
            addEntry(hash, key, value, i);
            return null;
        }

        // putForNullKey()的作用是将“key为null”键值对添加到table[0]位置
        private V putForNullKey(V value) {
            for (Entry<K,V> e = table[0]; e != null; e = e.next) {
                if (e.key == null) {
                    V oldValue = e.value;
                    e.value = value;
                    e.recordAccess(this);
                    return oldValue;
                }
            }
            // 这里的完全不会被执行到!
            modCount++;
            addEntry(0, null, value, 0);
            return null;
        }

        // 创建HashMap对应的“添加方法”，
        // 它和put()不同。putForCreate()是内部方法，它被构造函数等调用，用来创建HashMap
        // 而put()是对外提供的往HashMap中添加元素的方法。
        private void putForCreate(K key, V value) {
            int hash = (key == null) ? 0 : hash(key.hashCode());
            int i = indexFor(hash, table.length);

            // 若该HashMap表中存在“键值等于key”的元素，则替换该元素的value值
            for (Entry<K,V> e = table[i]; e != null; e = e.next) {
                Object k;
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    e.value = value;
                    return;
                }
            }

            // 若该HashMap表中不存在“键值等于key”的元素，则将该key-value添加到HashMap中
            createEntry(hash, key, value, i);
        }

        // 将“m”中的全部元素都添加到HashMap中。
        // 该方法被内部的构造HashMap的方法所调用。
        private void putAllForCreate(Map<? extends K, ? extends V> m) {
            // 利用迭代器将元素逐个添加到HashMap中
            for (Iterator<? extends Map.Entry<? extends K, ? extends V>> i = m.entrySet().iterator(); i.hasNext(); ) {
                Map.Entry<? extends K, ? extends V> e = i.next();
                putForCreate(e.getKey(), e.getValue());
            }
        }

        // 重新调整HashMap的大小，newCapacity是调整后的单位
        void resize(int newCapacity) {
            Entry[] oldTable = table;
            int oldCapacity = oldTable.length;
            if (oldCapacity == MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return;
            }

            // 新建一个HashMap，将“旧HashMap”的全部元素添加到“新HashMap”中，
            // 然后，将“新HashMap”赋值给“旧HashMap”。
            Entry[] newTable = new Entry[newCapacity];
            transfer(newTable);
            table = newTable;
            threshold = (int)(newCapacity * loadFactor);
        }

        // 将HashMap中的全部元素都添加到newTable中
        void transfer(Entry[] newTable) {
            Entry[] src = table;
            int newCapacity = newTable.length;
            for (int j = 0; j < src.length; j++) {
                Entry<K,V> e = src[j];
                if (e != null) {
                    src[j] = null;
                    do {
                        Entry<K,V> next = e.next;
                        int i = indexFor(e.hash, newCapacity);
                        e.next = newTable[i];
                        newTable[i] = e;
                        e = next;
                    } while (e != null);
                }
            }
        }

        // 将"m"的全部元素都添加到HashMap中
        public void putAll(Map<? extends K, ? extends V> m) {
            // 有效性判断
            int numKeysToBeAdded = m.size();
            if (numKeysToBeAdded == 0)
                return;

            // 计算容量是否足够，
            // 若“当前实际容量 < 需要的容量”，则将容量x2。
            if (numKeysToBeAdded > threshold) {
                int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
                if (targetCapacity > MAXIMUM_CAPACITY)
                    targetCapacity = MAXIMUM_CAPACITY;
                int newCapacity = table.length;
                while (newCapacity < targetCapacity)
                    newCapacity <<= 1;
                if (newCapacity > table.length)
                    resize(newCapacity);
            }

            // 通过迭代器，将“m”中的元素逐个添加到HashMap中。
            for (Iterator<? extends Map.Entry<? extends K, ? extends V>> i = m.entrySet().iterator(); i.hasNext(); ) {
                Map.Entry<? extends K, ? extends V> e = i.next();
                put(e.getKey(), e.getValue());
            }
        }

        // 删除“键为key”元素
        public V remove(Object key) {
            Entry<K,V> e = removeEntryForKey(key);
            return (e == null ? null : e.value);
        }

        // 删除“键为key”的元素
        final Entry<K,V> removeEntryForKey(Object key) {
            // 获取哈希值。若key为null，则哈希值为0；否则调用hash()进行计算
            int hash = (key == null) ? 0 : hash(key.hashCode());
            int i = indexFor(hash, table.length);
            Entry<K,V> prev = table[i];
            Entry<K,V> e = prev;

            // 删除链表中“键为key”的元素
            // 本质是“删除单向链表中的节点”
            while (e != null) {
                Entry<K,V> next = e.next;
                Object k;
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k)))) {
                    modCount++;
                    size--;
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    e.recordRemoval(this);
                    return e;
                }
                prev = e;
                e = next;
            }

            return e;
        }

        // 删除“键值对”
        final Entry<K,V> removeMapping(Object o) {
            if (!(o instanceof Map.Entry))
                return null;

            Map.Entry<K,V> entry = (Map.Entry<K,V>) o;
            Object key = entry.getKey();
            int hash = (key == null) ? 0 : hash(key.hashCode());
            int i = indexFor(hash, table.length);
            Entry<K,V> prev = table[i];
            Entry<K,V> e = prev;

            // 删除链表中的“键值对e”
            // 本质是“删除单向链表中的节点”
            while (e != null) {
                Entry<K,V> next = e.next;
                if (e.hash == hash && e.equals(entry)) {
                    modCount++;
                    size--;
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    e.recordRemoval(this);
                    return e;
                }
                prev = e;
                e = next;
            }

            return e;
        }

        // 清空HashMap，将所有的元素设为null
        public void clear() {
            modCount++;
            Entry[] tab = table;
            for (int i = 0; i < tab.length; i++)
                tab[i] = null;
            size = 0;
        }

        // 是否包含“值为value”的元素
        public boolean containsValue(Object value) {
        // 若“value为null”，则调用containsNullValue()查找
        if (value == null)
                return containsNullValue();

        // 若“value不为null”，则查找HashMap中是否有值为value的节点。
        Entry[] tab = table;
            for (int i = 0; i < tab.length ; i++)
                for (Entry e = tab[i] ; e != null ; e = e.next)
                    if (value.equals(e.value))
                        return true;
        return false;
        }

        // 是否包含null值
        private boolean containsNullValue() {
        Entry[] tab = table;
            for (int i = 0; i < tab.length ; i++)
                for (Entry e = tab[i] ; e != null ; e = e.next)
                    if (e.value == null)
                        return true;
        return false;
        }

        // 克隆一个HashMap，并返回Object对象
        public Object clone() {
            HashMap<K,V> result = null;
            try {
                result = (HashMap<K,V>)super.clone();
            } catch (CloneNotSupportedException e) {
                // assert false;
            }
            result.table = new Entry[table.length];
            result.entrySet = null;
            result.modCount = 0;
            result.size = 0;
            result.init();
            // 调用putAllForCreate()将全部元素添加到HashMap中
            result.putAllForCreate(this);

            return result;
        }

        // Entry是单向链表。
        // 它是 “HashMap链式存储法”对应的链表。
        // 它实现了Map.Entry 接口，即实现getKey(), getValue(), setValue(V value), equals(Object o), hashCode()这些函数
        static class Entry<K,V> implements Map.Entry<K,V> {
            final K key;
            V value;
            // 指向下一个节点
            Entry<K,V> next;
            final int hash;

            // 构造函数。
            // 输入参数包括"哈希值(h)", "键(k)", "值(v)", "下一节点(n)"
            Entry(int h, K k, V v, Entry<K,V> n) {
                value = v;
                next = n;
                key = k;
                hash = h;
            }

            public final K getKey() {
                return key;
            }

            public final V getValue() {
                return value;
            }

            public final V setValue(V newValue) {
                V oldValue = value;
                value = newValue;
                return oldValue;
            }

            // 判断两个Entry是否相等
            // 若两个Entry的“key”和“value”都相等，则返回true。
            // 否则，返回false
            public final boolean equals(Object o) {
                if (!(o instanceof Map.Entry))
                    return false;
                Map.Entry e = (Map.Entry)o;
                Object k1 = getKey();
                Object k2 = e.getKey();
                if (k1 == k2 || (k1 != null && k1.equals(k2))) {
                    Object v1 = getValue();
                    Object v2 = e.getValue();
                    if (v1 == v2 || (v1 != null && v1.equals(v2)))
                        return true;
                }
                return false;
            }

            // 实现hashCode()
            public final int hashCode() {
                return (key==null   ? 0 : key.hashCode()) ^
                       (value==null ? 0 : value.hashCode());
            }

            public final String toString() {
                return getKey() + "=" + getValue();
            }

            // 当向HashMap中添加元素时，绘调用recordAccess()。
            // 这里不做任何处理
            void recordAccess(HashMap<K,V> m) {
            }

            // 当从HashMap中删除元素时，绘调用recordRemoval()。
            // 这里不做任何处理
            void recordRemoval(HashMap<K,V> m) {
            }
        }

        // 新增Entry。将“key-value”插入指定位置，bucketIndex是位置索引。
        void addEntry(int hash, K key, V value, int bucketIndex) {
            // 保存“bucketIndex”位置的值到“e”中
            Entry<K,V> e = table[bucketIndex];
            // 设置“bucketIndex”位置的元素为“新Entry”，
            // 设置“e”为“新Entry的下一个节点”
            table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
            // 若HashMap的实际大小 不小于 “阈值”，则调整HashMap的大小
            if (size++ >= threshold)
                resize(2 * table.length);
        }

        // 创建Entry。将“key-value”插入指定位置，bucketIndex是位置索引。
        // 它和addEntry的区别是：
        // (01) addEntry()一般用在 新增Entry可能导致“HashMap的实际容量”超过“阈值”的情况下。
        //   例如，我们新建一个HashMap，然后不断通过put()向HashMap中添加元素；
        // put()是通过addEntry()新增Entry的。
        //   在这种情况下，我们不知道何时“HashMap的实际容量”会超过“阈值”；
        //   因此，需要调用addEntry()
        // (02) createEntry() 一般用在 新增Entry不会导致“HashMap的实际容量”超过“阈值”的情况下。
        //   例如，我们调用HashMap“带有Map”的构造函数，它绘将Map的全部元素添加到HashMap中；
        // 但在添加之前，我们已经计算好“HashMap的容量和阈值”。也就是，可以确定“即使将Map中
        // 的全部元素添加到HashMap中，都不会超过HashMap的阈值”。
        //   此时，调用createEntry()即可。
        void createEntry(int hash, K key, V value, int bucketIndex) {
            // 保存“bucketIndex”位置的值到“e”中
            Entry<K,V> e = table[bucketIndex];
            // 设置“bucketIndex”位置的元素为“新Entry”，
            // 设置“e”为“新Entry的下一个节点”
            table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
            size++;
        }

        // HashIterator是HashMap迭代器的抽象出来的父类，实现了公共了函数。
        // 它包含“key迭代器(KeyIterator)”、“Value迭代器(ValueIterator)”和“Entry迭代器(EntryIterator)”3个子类。
        private abstract class HashIterator<E> implements Iterator<E> {
            // 下一个元素
            Entry<K,V> next;
            // expectedModCount用于实现fast-fail机制。
            int expectedModCount;
            // 当前索引
            int index;
            // 当前元素
            Entry<K,V> current;

            HashIterator() {
                expectedModCount = modCount;
                if (size > 0) { // advance to first entry
                    Entry[] t = table;
                    // 将next指向table中第一个不为null的元素。
                    // 这里利用了index的初始值为0，从0开始依次向后遍历，直到找到不为null的元素就退出循环。
                    while (index < t.length && (next = t[index++]) == null)
                        ;
                }
            }

            public final boolean hasNext() {
                return next != null;
            }

            // 获取下一个元素
            final Entry<K,V> nextEntry() {
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                Entry<K,V> e = next;
                if (e == null)
                    throw new NoSuchElementException();

                // 注意！！！
                // 一个Entry就是一个单向链表
                // 若该Entry的下一个节点不为空，就将next指向下一个节点;
                // 否则，将next指向下一个链表(也是下一个Entry)的不为null的节点。
                if ((next = e.next) == null) {
                    Entry[] t = table;
                    while (index < t.length && (next = t[index++]) == null)
                        ;
                }
                current = e;
                return e;
            }

            // 删除当前元素
            public void remove() {
                if (current == null)
                    throw new IllegalStateException();
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
                Object k = current.key;
                current = null;
                HashMap.this.removeEntryForKey(k);
                expectedModCount = modCount;
            }

        }

        // value的迭代器
        private final class ValueIterator extends HashIterator<V> {
            public V next() {
                return nextEntry().value;
            }
        }

        // key的迭代器
        private final class KeyIterator extends HashIterator<K> {
            public K next() {
                return nextEntry().getKey();
            }
        }

        // Entry的迭代器
        private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
            public Map.Entry<K,V> next() {
                return nextEntry();
            }
        }

        // 返回一个“key迭代器”
        Iterator<K> newKeyIterator()   {
            return new KeyIterator();
        }
        // 返回一个“value迭代器”
        Iterator<V> newValueIterator()   {
            return new ValueIterator();
        }
        // 返回一个“entry迭代器”
        Iterator<Map.Entry<K,V>> newEntryIterator()   {
            return new EntryIterator();
        }

        // HashMap的Entry对应的集合
        private transient Set<Map.Entry<K,V>> entrySet = null;

        // 返回“key的集合”，实际上返回一个“KeySet对象”
        public Set<K> keySet() {
            Set<K> ks = keySet;
            return (ks != null ? ks : (keySet = new KeySet()));
        }

        // Key对应的集合
        // KeySet继承于AbstractSet，说明该集合中没有重复的Key。
        private final class KeySet extends AbstractSet<K> {
            public Iterator<K> iterator() {
                return newKeyIterator();
            }
            public int size() {
                return size;
            }
            public boolean contains(Object o) {
                return containsKey(o);
            }
            public boolean remove(Object o) {
                return HashMap.this.removeEntryForKey(o) != null;
            }
            public void clear() {
                HashMap.this.clear();
            }
        }

        // 返回“value集合”，实际上返回的是一个Values对象
        public Collection<V> values() {
            Collection<V> vs = values;
            return (vs != null ? vs : (values = new Values()));
        }

        // “value集合”
        // Values继承于AbstractCollection，不同于“KeySet继承于AbstractSet”，
        // Values中的元素能够重复。因为不同的key可以指向相同的value。
        private final class Values extends AbstractCollection<V> {
            public Iterator<V> iterator() {
                return newValueIterator();
            }
            public int size() {
                return size;
            }
            public boolean contains(Object o) {
                return containsValue(o);
            }
            public void clear() {
                HashMap.this.clear();
            }
        }

        // 返回“HashMap的Entry集合”
        public Set<Map.Entry<K,V>> entrySet() {
            return entrySet0();
        }

        // 返回“HashMap的Entry集合”，它实际是返回一个EntrySet对象
        private Set<Map.Entry<K,V>> entrySet0() {
            Set<Map.Entry<K,V>> es = entrySet;
            return es != null ? es : (entrySet = new EntrySet());
        }

        // EntrySet对应的集合
        // EntrySet继承于AbstractSet，说明该集合中没有重复的EntrySet。
        private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
            public Iterator<Map.Entry<K,V>> iterator() {
                return newEntryIterator();
            }
            public boolean contains(Object o) {
                if (!(o instanceof Map.Entry))
                    return false;
                Map.Entry<K,V> e = (Map.Entry<K,V>) o;
                Entry<K,V> candidate = getEntry(e.getKey());
                return candidate != null && candidate.equals(e);
            }
            public boolean remove(Object o) {
                return removeMapping(o) != null;
            }
            public int size() {
                return size;
            }
            public void clear() {
                HashMap.this.clear();
            }
        }

        // java.io.Serializable的写入函数
        // 将HashMap的“总的容量，实际容量，所有的Entry”都写入到输出流中
        private void writeObject(java.io.ObjectOutputStream s)
            throws IOException
        {
            Iterator<Map.Entry<K,V>> i =
                (size > 0) ? entrySet0().iterator() : null;

            // Write out the threshold, loadfactor, and any hidden stuff
            s.defaultWriteObject();

            // Write out number of buckets
            s.writeInt(table.length);

            // Write out size (number of Mappings)
            s.writeInt(size);

            // Write out keys and values (alternating)
            if (i != null) {
                while (i.hasNext()) {
                Map.Entry<K,V> e = i.next();
                s.writeObject(e.getKey());
                s.writeObject(e.getValue());
                }
            }
        }


        private static final long serialVersionUID = 362498820763181265L;

        // java.io.Serializable的读取函数：根据写入方式读出
        // 将HashMap的“总的容量，实际容量，所有的Entry”依次读出
        private void readObject(java.io.ObjectInputStream s)
             throws IOException, ClassNotFoundException
        {
            // Read in the threshold, loadfactor, and any hidden stuff
            s.defaultReadObject();

            // Read in number of buckets and allocate the bucket array;
            int numBuckets = s.readInt();
            table = new Entry[numBuckets];

            init();  // Give subclass a chance to do its thing.

            // Read in size (number of Mappings)
            int size = s.readInt();

            // Read the keys and values, and put the mappings in the HashMap
            for (int i=0; i<size; i++) {
                K key = (K) s.readObject();
                V value = (V) s.readObject();
                putForCreate(key, value);
            }
        }

        // 返回“HashMap总的容量”
        int   capacity()     { return table.length; }
        // 返回“HashMap的加载因子”
        float loadFactor()   { return loadFactor;   }
    }

说明:  
在详细介绍HashMap的代码之前，我们需要了解：HashMap就是一个散列表，它是通过“拉链法”解决哈希冲突的。  
还需要再补充说明的一点是影响HashMap性能的有两个参数：初始容量(initialCapacity) 和加载因子(loadFactor)。容量 是哈希表中桶的数量，初始容量只是哈希表在创建时的容量。加载因子 是哈希表在其容量自动增加之前可以达到多满的一种尺度。当哈希表中的条目数超出了加载因子与当前容量的乘积时，则要对该哈希表进行 rehash 操作（即重建内部数据结构），从而哈希表将具有大约两倍的桶数。


 
<a name="anchor3_1"></a>
## 第3.1部分 HashMap的“拉链法”相关内容

### 3.1.1 HashMap数据存储数组

    transient Entry[] table;

HashMap中的key-value都是存储在Entry数组中的。

### 3.1.2 数据节点Entry的数据结构

    static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        // 指向下一个节点
        Entry<K,V> next;
        final int hash;

        // 构造函数。
        // 输入参数包括"哈希值(h)", "键(k)", "值(v)", "下一节点(n)"
        Entry(int h, K k, V v, Entry<K,V> n) {
            value = v;
            next = n;
            key = k;
            hash = h;
        }

        public final K getKey() {
            return key;
        }

        public final V getValue() {
            return value;
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        // 判断两个Entry是否相等
        // 若两个Entry的“key”和“value”都相等，则返回true。
        // 否则，返回false
        public final boolean equals(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry e = (Map.Entry)o;
            Object k1 = getKey();
            Object k2 = e.getKey();
            if (k1 == k2 || (k1 != null && k1.equals(k2))) {
                Object v1 = getValue();
                Object v2 = e.getValue();
                if (v1 == v2 || (v1 != null && v1.equals(v2)))
                    return true;
            }
            return false;
        }

        // 实现hashCode()
        public final int hashCode() {
            return (key==null   ? 0 : key.hashCode()) ^
                   (value==null ? 0 : value.hashCode());
        }

        public final String toString() {
            return getKey() + "=" + getValue();
        }

        // 当向HashMap中添加元素时，绘调用recordAccess()。
        // 这里不做任何处理
        void recordAccess(HashMap<K,V> m) {
        }

        // 当从HashMap中删除元素时，绘调用recordRemoval()。
        // 这里不做任何处理
        void recordRemoval(HashMap<K,V> m) {
        }
    }

从中，我们可以看出 Entry 实际上就是一个单向链表。这也是为什么我们说HashMap是通过拉链法解决哈希冲突的。  
Entry 实现了Map.Entry 接口，即实现getKey(), getValue(), setValue(V value), equals(Object o), hashCode()这些函数。这些都是基本的读取/修改key、value值的函数。

 

 
<a name="anchor3_2"></a>
## 第3.2部分 HashMap的构造函数

HashMap共包括4个构造函数

    // 默认构造函数。
    public HashMap() {
        // 设置“加载因子”
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        // 设置“HashMap阈值”，当HashMap中存储数据的数量达到threshold时，就需要将HashMap的容量加倍。
        threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
        // 创建Entry数组，用来保存数据
        table = new Entry[DEFAULT_INITIAL_CAPACITY];
        init();
    }

    // 指定“容量大小”和“加载因子”的构造函数
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        // HashMap的最大容量只能是MAXIMUM_CAPACITY
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        // Find a power of 2 >= initialCapacity
        int capacity = 1;
        while (capacity < initialCapacity)
            capacity <<= 1;

        // 设置“加载因子”
        this.loadFactor = loadFactor;
        // 设置“HashMap阈值”，当HashMap中存储数据的数量达到threshold时，就需要将HashMap的容量加倍。
        threshold = (int)(capacity * loadFactor);
        // 创建Entry数组，用来保存数据
        table = new Entry[capacity];
        init();
    }

    // 指定“容量大小”的构造函数
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    // 包含“子Map”的构造函数
    public HashMap(Map<? extends K, ? extends V> m) {
        this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                      DEFAULT_INITIAL_CAPACITY), DEFAULT_LOAD_FACTOR);
        // 将m中的全部元素逐个添加到HashMap中
        putAllForCreate(m);
    }
 

 
<a name="anchor3_3"></a>
## 第3.3部分 HashMap的主要对外接口

### 3.3.1 clear()

clear() 的作用是清空HashMap。它是通过将所有的元素设为null来实现的。

    public void clear() {
        modCount++;
        Entry[] tab = table;
        for (int i = 0; i < tab.length; i++)
            tab[i] = null;
        size = 0;
    }
 

### 3.3.2 containsKey()

containsKey() 的作用是判断HashMap是否包含key。

    public boolean containsKey(Object key) {
        return getEntry(key) != null;
    }

containsKey() 首先通过getEntry(key)获取key对应的Entry，然后判断该Entry是否为null。  
getEntry()的源码如下：  

    final Entry<K,V> getEntry(Object key) {
        // 获取哈希值
        // HashMap将“key为null”的元素存储在table[0]位置，“key不为null”的则调用hash()计算哈希值
        int hash = (key == null) ? 0 : hash(key.hashCode());
        // 在“该hash值对应的链表”上查找“键值等于key”的元素
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }

getEntry() 的作用就是返回“键为key”的键值对，它的实现源码中已经进行了说明。  
这里需要强调的是：HashMap将“key为null”的元素都放在table的位置0处，即table[0]中；“key不为null”的放在table的其余位置！


### 3.3.3 containsValue()

containsValue() 的作用是判断HashMap是否包含“值为value”的元素。

    public boolean containsValue(Object value) {
        // 若“value为null”，则调用containsNullValue()查找
        if (value == null)
            return containsNullValue();

        // 若“value不为null”，则查找HashMap中是否有值为value的节点。
        Entry[] tab = table;
        for (int i = 0; i < tab.length ; i++)
            for (Entry e = tab[i] ; e != null ; e = e.next)
                if (value.equals(e.value))
                    return true;
        return false;
    }

从中，我们可以看出containsNullValue()分为两步进行处理：第一，若“value为null”，则调用containsNullValue()。第二，若“value不为null”，则查找HashMap中是否有值为value的节点。

containsNullValue() 的作用判断HashMap中是否包含“值为null”的元素。

    private boolean containsNullValue() {
        Entry[] tab = table;
        for (int i = 0; i < tab.length ; i++)
            for (Entry e = tab[i] ; e != null ; e = e.next)
                if (e.value == null)
                    return true;
        return false;
    }
 

### 3.3.4 entrySet()、values()、keySet()

它们3个的原理类似，这里以entrySet()为例来说明。  
entrySet()的作用是返回“HashMap中所有Entry的集合”，它是一个集合。实现代码如下：

    // 返回“HashMap的Entry集合”
    public Set<Map.Entry<K,V>> entrySet() {
        return entrySet0();
    }

    // 返回“HashMap的Entry集合”，它实际是返回一个EntrySet对象
    private Set<Map.Entry<K,V>> entrySet0() {
        Set<Map.Entry<K,V>> es = entrySet;
        return es != null ? es : (entrySet = new EntrySet());
    }

    // EntrySet对应的集合
    // EntrySet继承于AbstractSet，说明该集合中没有重复的EntrySet。
    private final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public Iterator<Map.Entry<K,V>> iterator() {
            return newEntryIterator();
        }
        public boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<K,V> e = (Map.Entry<K,V>) o;
            Entry<K,V> candidate = getEntry(e.getKey());
            return candidate != null && candidate.equals(e);
        }
        public boolean remove(Object o) {
            return removeMapping(o) != null;
        }
        public int size() {
            return size;
        }
        public void clear() {
            HashMap.this.clear();
        }
    }
 

HashMap是通过拉链法实现的散列表。表现在HashMap包括许多的Entry，而每一个Entry本质上又是一个单向链表。那么HashMap遍历key-value键值对的时候，是如何逐个去遍历的呢？


下面我们就看看HashMap是如何通过entrySet()遍历的。  
entrySet()实际上是通过newEntryIterator()实现的。 下面我们看看它的代码：

    // 返回一个“entry迭代器”
    Iterator<Map.Entry<K,V>> newEntryIterator()   {
        return new EntryIterator();
    }

    // Entry的迭代器
    private final class EntryIterator extends HashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() {
            return nextEntry();
        }
    }

    // HashIterator是HashMap迭代器的抽象出来的父类，实现了公共了函数。
    // 它包含“key迭代器(KeyIterator)”、“Value迭代器(ValueIterator)”和“Entry迭代器(EntryIterator)”3个子类。
    private abstract class HashIterator<E> implements Iterator<E> {
        // 下一个元素
        Entry<K,V> next;
        // expectedModCount用于实现fast-fail机制。
        int expectedModCount;
        // 当前索引
        int index;
        // 当前元素
        Entry<K,V> current;

        HashIterator() {
            expectedModCount = modCount;
            if (size > 0) { // advance to first entry
                Entry[] t = table;
                // 将next指向table中第一个不为null的元素。
                // 这里利用了index的初始值为0，从0开始依次向后遍历，直到找到不为null的元素就退出循环。
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        // 获取下一个元素
        final Entry<K,V> nextEntry() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Entry<K,V> e = next;
            if (e == null)
                throw new NoSuchElementException();

            // 注意！！！
            // 一个Entry就是一个单向链表
            // 若该Entry的下一个节点不为空，就将next指向下一个节点;
            // 否则，将next指向下一个链表(也是下一个Entry)的不为null的节点。
            if ((next = e.next) == null) {
                Entry[] t = table;
                while (index < t.length && (next = t[index++]) == null)
                    ;
            }
            current = e;
            return e;
        }

        // 删除当前元素
        public void remove() {
            if (current == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            Object k = current.key;
            current = null;
            HashMap.this.removeEntryForKey(k);
            expectedModCount = modCount;
        }

    }

当我们通过entrySet()获取到的Iterator的next()方法去遍历HashMap时，实际上调用的是 nextEntry() 。而nextEntry()的实现方式，先遍历Entry(根据Entry在table中的序号，从小到大的遍历)；然后对每个Entry(即每个单向链表)，逐个遍历。


### 3.3.5 get()

get() 的作用是获取key对应的value，它的实现代码如下：

    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        // 获取key的hash值
        int hash = hash(key.hashCode());
        // 在“该hash值对应的链表”上查找“键值等于key”的元素
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
    }
 

### 3.3.6 put()

put() 的作用是对外提供接口，让HashMap对象可以通过put()将“key-value”添加到HashMap中。

    public V put(K key, V value) {
        // 若“key为null”，则将该键值对添加到table[0]中。
        if (key == null)
            return putForNullKey(value);
        // 若“key不为null”，则计算该key的哈希值，然后将其添加到该哈希值对应的链表中。
        int hash = hash(key.hashCode());
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            // 若“该key”对应的键值对已经存在，则用新的value取代旧的value。然后退出！
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        // 若“该key”对应的键值对不存在，则将“key-value”添加到table中
        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }

若要添加到HashMap中的键值对对应的key已经存在HashMap中，则找到该键值对；然后新的value取代旧的value，并退出！
若要添加到HashMap中的键值对对应的key不在HashMap中，则将其添加到该哈希值对应的链表中，并调用addEntry()。
下面看看addEntry()的代码：

    void addEntry(int hash, K key, V value, int bucketIndex) {
        // 保存“bucketIndex”位置的值到“e”中
        Entry<K,V> e = table[bucketIndex];
        // 设置“bucketIndex”位置的元素为“新Entry”，
        // 设置“e”为“新Entry的下一个节点”
        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
        // 若HashMap的实际大小 不小于 “阈值”，则调整HashMap的大小
        if (size++ >= threshold)
            resize(2 * table.length);
    }

addEntry() 的作用是新增Entry。将“key-value”插入指定位置，bucketIndex是位置索引。

说到addEntry()，就不得不说另一个函数createEntry()。createEntry()的代码如下：

    void createEntry(int hash, K key, V value, int bucketIndex) {
        // 保存“bucketIndex”位置的值到“e”中
        Entry<K,V> e = table[bucketIndex];
        // 设置“bucketIndex”位置的元素为“新Entry”，
        // 设置“e”为“新Entry的下一个节点”
        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
        size++;
    }

它们的作用都是将key、value添加到HashMap中。而且，比较addEntry()和createEntry()的代码，我们发现addEntry()多了两句：

    if (size++ >= threshold)
        resize(2 * table.length);

那它们的区别到底是什么呢？  
阅读代码，我们可以发现，它们的使用情景不同。

(01) addEntry()一般用在 新增Entry可能导致“HashMap的实际容量”超过“阈值”的情况下。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，我们新建一个HashMap，然后不断通过put()向HashMap中添加元素；put()是通过addEntry()新增Entry的。  
&nbsp;&nbsp;&nbsp;&nbsp; 在这种情况下，我们不知道何时“HashMap的实际容量”会超过“阈值”；  
&nbsp;&nbsp;&nbsp;&nbsp; 因此，需要调用addEntry()。  
(02) createEntry() 一般用在 新增Entry不会导致“HashMap的实际容量”超过“阈值”的情况下。  
&nbsp;&nbsp;&nbsp;&nbsp; 例如，我们调用HashMap“带有Map”的构造函数，它绘将Map的全部元素添加到HashMap中；  
&nbsp;&nbsp;&nbsp;&nbsp; 但在添加之前，我们已经计算好“HashMap的容量和阈值”。也就是，可以确定“即使将Map中的全部元素添加到HashMap中，都不会超过HashMap的阈值”。  
此时，调用createEntry()即可。

 

### 3.3.7 putAll()

putAll() 的作用是将"m"的全部元素都添加到HashMap中，它的代码如下：

    public void putAll(Map<? extends K, ? extends V> m) {
        // 有效性判断
        int numKeysToBeAdded = m.size();
        if (numKeysToBeAdded == 0)
            return;

        // 计算容量是否足够，
        // 若“当前实际容量 < 需要的容量”，则将容量x2。
        if (numKeysToBeAdded > threshold) {
            int targetCapacity = (int)(numKeysToBeAdded / loadFactor + 1);
            if (targetCapacity > MAXIMUM_CAPACITY)
                targetCapacity = MAXIMUM_CAPACITY;
            int newCapacity = table.length;
            while (newCapacity < targetCapacity)
                newCapacity <<= 1;
            if (newCapacity > table.length)
                resize(newCapacity);
        }

        // 通过迭代器，将“m”中的元素逐个添加到HashMap中。
        for (Iterator<? extends Map.Entry<? extends K, ? extends V>> i = m.entrySet().iterator(); i.hasNext(); ) {
            Map.Entry<? extends K, ? extends V> e = i.next();
            put(e.getKey(), e.getValue());
        }
    }
 

3.3.8 remove()

remove() 的作用是删除“键为key”元素

    public V remove(Object key) {
        Entry<K,V> e = removeEntryForKey(key);
        return (e == null ? null : e.value);
    }


    // 删除“键为key”的元素
    final Entry<K,V> removeEntryForKey(Object key) {
        // 获取哈希值。若key为null，则哈希值为0；否则调用hash()进行计算
        int hash = (key == null) ? 0 : hash(key.hashCode());
        int i = indexFor(hash, table.length);
        Entry<K,V> prev = table[i];
        Entry<K,V> e = prev;

        // 删除链表中“键为key”的元素
        // 本质是“删除单向链表中的节点”
        while (e != null) {
            Entry<K,V> next = e.next;
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k)))) {
                modCount++;
                size--;
                if (prev == e)
                    table[i] = next;
                else
                    prev.next = next;
                e.recordRemoval(this);
                return e;
            }
            prev = e;
            e = next;
        }

        return e;
    }
 

 
<a name="anchor3_4"></a>
# 第3.4部分 HashMap实现的Cloneable接口

HashMap实现了Cloneable接口，即实现了clone()方法。  
clone()方法的作用很简单，就是克隆一个HashMap对象并返回。

    // 克隆一个HashMap，并返回Object对象
    public Object clone() {
        HashMap<K,V> result = null;
        try {
            result = (HashMap<K,V>)super.clone();
        } catch (CloneNotSupportedException e) {
            // assert false;
        }
        result.table = new Entry[table.length];
        result.entrySet = null;
        result.modCount = 0;
        result.size = 0;
        result.init();
        // 调用putAllForCreate()将全部元素添加到HashMap中
        result.putAllForCreate(this);

        return result;
    }

 
<a name="anchor3_5"></a>
# 第3.5部分 HashMap实现的Serializable接口

HashMap实现java.io.Serializable，分别实现了串行读取、写入功能。  
串行写入函数是writeObject()，它的作用是将HashMap的“总的容量，实际容量，所有的Entry”都写入到输出流中。  
而串行读取函数是readObject()，它的作用是将HashMap的“总的容量，实际容量，所有的Entry”依次读出

    // java.io.Serializable的写入函数
    // 将HashMap的“总的容量，实际容量，所有的Entry”都写入到输出流中
    private void writeObject(java.io.ObjectOutputStream s)
        throws IOException
    {
        Iterator<Map.Entry<K,V>> i =
            (size > 0) ? entrySet0().iterator() : null;

        // Write out the threshold, loadfactor, and any hidden stuff
        s.defaultWriteObject();

        // Write out number of buckets
        s.writeInt(table.length);

        // Write out size (number of Mappings)
        s.writeInt(size);

        // Write out keys and values (alternating)
        if (i != null) {
            while (i.hasNext()) {
            Map.Entry<K,V> e = i.next();
            s.writeObject(e.getKey());
            s.writeObject(e.getValue());
            }
        }
    }

    // java.io.Serializable的读取函数：根据写入方式读出
    // 将HashMap的“总的容量，实际容量，所有的Entry”依次读出
    private void readObject(java.io.ObjectInputStream s)
         throws IOException, ClassNotFoundException
    {
        // Read in the threshold, loadfactor, and any hidden stuff
        s.defaultReadObject();

        // Read in number of buckets and allocate the bucket array;
        int numBuckets = s.readInt();
        table = new Entry[numBuckets];

        init();  // Give subclass a chance to do its thing.

        // Read in size (number of Mappings)
        int size = s.readInt();

        // Read the keys and values, and put the mappings in the HashMap
        for (int i=0; i<size; i++) {
            K key = (K) s.readObject();
            V value = (V) s.readObject();
            putForCreate(key, value);
        }
    }

 
 
<a name="anchor4"></a>
# 第4部分 HashMap遍历方式

## 4.1 遍历HashMap的键值对

第一步：根据entrySet()获取HashMap的“键值对”的Set集合。

第二步：通过Iterator迭代器遍历“第一步”得到的集合。

    // 假设map是HashMap对象
    // map中的key是String类型，value是Integer类型
    Integer integ = null;
    Iterator iter = map.entrySet().iterator();
    while(iter.hasNext()) {
        Map.Entry entry = (Map.Entry)iter.next();
        // 获取key
        key = (String)entry.getKey();
            // 获取value
        integ = (Integer)entry.getValue();
    }

## 4.2 遍历HashMap的键

第一步：根据keySet()获取HashMap的“键”的Set集合。

第二步：通过Iterator迭代器遍历“第一步”得到的集合。

    // 假设map是HashMap对象
    // map中的key是String类型，value是Integer类型
    String key = null;
    Integer integ = null;
    Iterator iter = map.keySet().iterator();
    while (iter.hasNext()) {
            // 获取key
        key = (String)iter.next();
            // 根据key，获取value
        integ = (Integer)map.get(key);
    }

## 4.3 遍历HashMap的值

第一步：根据value()获取HashMap的“值”的集合。

第二步：通过Iterator迭代器遍历“第一步”得到的集合。

    // 假设map是HashMap对象
    // map中的key是String类型，value是Integer类型
    Integer value = null;
    Collection c = map.values();
    Iterator iter= c.iterator();
    while (iter.hasNext()) {
        value = (Integer)iter.next();
    }

 

遍历测试程序如下：

    import java.util.Map;
    import java.util.Random;
    import java.util.Iterator;
    import java.util.HashMap;
    import java.util.HashSet;
    import java.util.Map.Entry;
    import java.util.Collection;

    /*
     * @desc 遍历HashMap的测试程序。
     *   (01) 通过entrySet()去遍历key、value，参考实现函数：
     *        iteratorHashMapByEntryset()
     *   (02) 通过keySet()去遍历key、value，参考实现函数：
     *        iteratorHashMapByKeyset()
     *   (03) 通过values()去遍历value，参考实现函数：
     *        iteratorHashMapJustValues()
     *
     * @author skywang
     */
    public class HashMapIteratorTest {

        public static void main(String[] args) {
            int val = 0;
            String key = null;
            Integer value = null;
            Random r = new Random();
            HashMap map = new HashMap();

            for (int i=0; i<12; i++) {
                // 随机获取一个[0,100)之间的数字
                val = r.nextInt(100);
                
                key = String.valueOf(val);
                value = r.nextInt(5);
                // 添加到HashMap中
                map.put(key, value);
                System.out.println(" key:"+key+" value:"+value);
            }
            // 通过entrySet()遍历HashMap的key-value
            iteratorHashMapByEntryset(map) ;
            
            // 通过keySet()遍历HashMap的key-value
            iteratorHashMapByKeyset(map) ;
            
            // 单单遍历HashMap的value
            iteratorHashMapJustValues(map);        
        }
        
        /*
         * 通过entry set遍历HashMap
         * 效率高!
         */
        private static void iteratorHashMapByEntryset(HashMap map) {
            if (map == null)
                return ;

            System.out.println("\niterator HashMap By entryset");
            String key = null;
            Integer integ = null;
            Iterator iter = map.entrySet().iterator();
            while(iter.hasNext()) {
                Map.Entry entry = (Map.Entry)iter.next();
                
                key = (String)entry.getKey();
                integ = (Integer)entry.getValue();
                System.out.println(key+" -- "+integ.intValue());
            }
        }

        /*
         * 通过keyset来遍历HashMap
         * 效率低!
         */
        private static void iteratorHashMapByKeyset(HashMap map) {
            if (map == null)
                return ;

            System.out.println("\niterator HashMap By keyset");
            String key = null;
            Integer integ = null;
            Iterator iter = map.keySet().iterator();
            while (iter.hasNext()) {
                key = (String)iter.next();
                integ = (Integer)map.get(key);
                System.out.println(key+" -- "+integ.intValue());
            }
        }
        

        /*
         * 遍历HashMap的values
         */
        private static void iteratorHashMapJustValues(HashMap map) {
            if (map == null)
                return ;
            
            Collection c = map.values();
            Iterator iter= c.iterator();
            while (iter.hasNext()) {
                System.out.println(iter.next());
           }
        }
    }
 
 
<a name="anchor5"></a>
# 第5部分 HashMap示例

下面通过一个实例学习如何使用HashMap

    import java.util.Map;
    import java.util.Random;
    import java.util.Iterator;
    import java.util.HashMap;
    import java.util.HashSet;
    import java.util.Map.Entry;
    import java.util.Collection;

    /*
     * @desc HashMap测试程序
     *        
     * @author skywang
     */
    public class HashMapTest {

        public static void main(String[] args) {
            testHashMapAPIs();
        }
        
        private static void testHashMapAPIs() {
            // 初始化随机种子
            Random r = new Random();
            // 新建HashMap
            HashMap map = new HashMap();
            // 添加操作
            map.put("one", r.nextInt(10));
            map.put("two", r.nextInt(10));
            map.put("three", r.nextInt(10));

            // 打印出map
            System.out.println("map:"+map );

            // 通过Iterator遍历key-value
            Iterator iter = map.entrySet().iterator();
            while(iter.hasNext()) {
                Map.Entry entry = (Map.Entry)iter.next();
                System.out.println("next : "+ entry.getKey() +" - "+entry.getValue());
            }

            // HashMap的键值对个数        
            System.out.println("size:"+map.size());

            // containsKey(Object key) :是否包含键key
            System.out.println("contains key two : "+map.containsKey("two"));
            System.out.println("contains key five : "+map.containsKey("five"));

            // containsValue(Object value) :是否包含值value
            System.out.println("contains value 0 : "+map.containsValue(new Integer(0)));

            // remove(Object key) ： 删除键key对应的键值对
            map.remove("three");

            System.out.println("map:"+map );

            // clear() ： 清空HashMap
            map.clear();

            // isEmpty() : HashMap是否为空
            System.out.println((map.isEmpty()?"map is empty":"map is not empty") );
        }
    }

 (某一次)运行结果： 

    map:{two=7, one=9, three=6}
    next : two - 7
    next : one - 9
    next : three - 6
    size:3
    contains key two : true
    contains key five : false
    contains value 0 : false
    map:{two=7, one=9}
    map is empty


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
