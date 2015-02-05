---
layout: post
title: "Java 集合系列17之 TreeSet详细介绍(源码解析)和使用示例"
description: "java collection"
category: java
tags: [java]
date: 2012-02-17 09:01
---

 
> 这一章，我们对TreeSet进行学习。  
> 我们先对TreeSet有个整体认识，然后再学习它的源码，最后再通过实例来学会使用TreeSet。

> **目录**  
> [第1部分 TreeSet介绍](#anchor1)   
> [第2部分 TreeSet数据结构](#anchor2)   
> [第3部分 TreeSet源码解析(基于JDK1.6.0_45)](#anchor3)   
> [第4部分 TreeSet遍历方式](#anchor4)   
> [第5部分 TreeSet示例](#anchor5)   

转载请注明出处：http://www.cnblogs.com/skywang12345/admin/EditPosts.aspx?postid=3311268

 
 
<a name="anchor1"></a>
# 第1部分 TreeSet介绍

## TreeSet简介

TreeSet 是一个有序的集合，它的作用是提供有序的Set集合。它继承于AbstractSet抽象类，实现了NavigableSet<E>, Cloneable, java.io.Serializable接口。  
TreeSet 继承于AbstractSet，所以它是一个Set集合，具有Set的属性和方法。  
TreeSet 实现了NavigableSet接口，意味着它支持一系列的导航方法。比如查找与指定目标最匹配项。  
TreeSet 实现了Cloneable接口，意味着它能被克隆。  
TreeSet 实现了java.io.Serializable接口，意味着它支持序列化。

TreeSet是基于TreeMap实现的。TreeSet中的元素支持2种排序方式：自然排序 或者 根据创建TreeSet 时提供的 Comparator 进行排序。这取决于使用的构造方法。  
TreeSet为基本操作（add、remove 和 contains）提供受保证的 log(n) 时间开销。  
另外，TreeSet是非同步的。 它的iterator 方法返回的迭代器是fail-fast的。

 

## TreeSet的构造函数

    // 默认构造函数。使用该构造函数，TreeSet中的元素按照自然排序进行排列。
    TreeSet()

    // 创建的TreeSet包含collection
    TreeSet(Collection<? extends E> collection)

    // 指定TreeSet的比较器
    TreeSet(Comparator<? super E> comparator)

    // 创建的TreeSet包含set
    TreeSet(SortedSet<E> set)

TreeSet的API

    boolean                   add(E object)
    boolean                   addAll(Collection<? extends E> collection)
    void                      clear()
    Object                    clone()
    boolean                   contains(Object object)
    E                         first()
    boolean                   isEmpty()
    E                         last()
    E                         pollFirst()
    E                         pollLast()
    E                         lower(E e)
    E                         floor(E e)
    E                         ceiling(E e)
    E                         higher(E e)
    boolean                   remove(Object object)
    int                       size()
    Comparator<? super E>     comparator()
    Iterator<E>               iterator()
    Iterator<E>               descendingIterator()
    SortedSet<E>              headSet(E end)
    NavigableSet<E>           descendingSet()
    NavigableSet<E>           headSet(E end, boolean endInclusive)
    SortedSet<E>              subSet(E start, E end)
    NavigableSet<E>           subSet(E start, boolean startInclusive, E end, boolean endInclusive)
    NavigableSet<E>           tailSet(E start, boolean startInclusive)
    SortedSet<E>              tailSet(E start)

说明：

(01) TreeSet是有序的Set集合，因此支持add、remove、get等方法。  
(02) 和NavigableSet一样，TreeSet的导航方法大致可以区分为两类，一类时提供元素项的导航方法，返回某个元素；另一类时提供集合的导航方法，返回某个集合。  
lower、floor、ceiling 和 higher 分别返回小于、小于等于、大于等于、大于给定元素的元素，如果不存在这样的元素，则返回 null。

 
 
<a name="anchor2"></a>
# 第2部分 TreeSet数据结构

TreeSet的继承关系

    java.lang.Object
       ↳     java.util.AbstractCollection<E>
             ↳     java.util.AbstractSet<E>
                   ↳     java.util.TreeSet<E>

TreeSet的声明

    public class TreeSet<E> extends AbstractSet<E>        
    implements NavigableSet<E>, Cloneable, java.io.Serializable{}

 

TreeSet与Collection关系如下图：

![img](/media/pic/java/collection/collection17.jpg)

从图中可以看出：  
(01) TreeSet继承于AbstractSet，并且实现了NavigableSet接口。  
(02) TreeSet的本质是一个"有序的，并且没有重复元素"的集合，它是通过TreeMap实现的。TreeSet中含有一个"NavigableMap类型的成员变量"m，而m实际上是"TreeMap的实例"。

 
 
<a name="anchor3"></a>
# 第3部分 TreeSet源码解析(基于JDK1.6.0_45)

为了更了解TreeSet的原理，下面对TreeSet源码代码作出分析。

    package java.util;

    public class TreeSet<E> extends AbstractSet<E>
        implements NavigableSet<E>, Cloneable, java.io.Serializable
    {
        // NavigableMap对象
        private transient NavigableMap<E,Object> m;

        // TreeSet是通过TreeMap实现的，
        // PRESENT是键-值对中的值。
        private static final Object PRESENT = new Object();

        // 不带参数的构造函数。创建一个空的TreeMap
        public TreeSet() {
            this(new TreeMap<E,Object>());
        }

        // 将TreeMap赋值给 "NavigableMap对象m"
        TreeSet(NavigableMap<E,Object> m) {
            this.m = m;
        }

        // 带比较器的构造函数。
        public TreeSet(Comparator<? super E> comparator) {
            this(new TreeMap<E,Object>(comparator));
        }

        // 创建TreeSet，并将集合c中的全部元素都添加到TreeSet中
        public TreeSet(Collection<? extends E> c) {
            this();
            // 将集合c中的元素全部添加到TreeSet中
            addAll(c);
        }

        // 创建TreeSet，并将s中的全部元素都添加到TreeSet中
        public TreeSet(SortedSet<E> s) {
            this(s.comparator());
            addAll(s);
        }

        // 返回TreeSet的顺序排列的迭代器。
        // 因为TreeSet时TreeMap实现的，所以这里实际上时返回TreeMap的“键集”对应的迭代器
        public Iterator<E> iterator() {
            return m.navigableKeySet().iterator();
        }

        // 返回TreeSet的逆序排列的迭代器。
        // 因为TreeSet时TreeMap实现的，所以这里实际上时返回TreeMap的“键集”对应的迭代器
        public Iterator<E> descendingIterator() {
            return m.descendingKeySet().iterator();
        }

        // 返回TreeSet的大小
        public int size() {
            return m.size();
        }

        // 返回TreeSet是否为空
        public boolean isEmpty() {
            return m.isEmpty();
        }

        // 返回TreeSet是否包含对象(o)
        public boolean contains(Object o) {
            return m.containsKey(o);
        }

        // 添加e到TreeSet中
        public boolean add(E e) {
            return m.put(e, PRESENT)==null;
        }

        // 删除TreeSet中的对象o
        public boolean remove(Object o) {
            return m.remove(o)==PRESENT;
        }

        // 清空TreeSet
        public void clear() {
            m.clear();
        }

        // 将集合c中的全部元素添加到TreeSet中
        public  boolean addAll(Collection<? extends E> c) {
            // Use linear-time version if applicable
            if (m.size()==0 && c.size() > 0 &&
                c instanceof SortedSet &&
                m instanceof TreeMap) {
                SortedSet<? extends E> set = (SortedSet<? extends E>) c;
                TreeMap<E,Object> map = (TreeMap<E, Object>) m;
                Comparator<? super E> cc = (Comparator<? super E>) set.comparator();
                Comparator<? super E> mc = map.comparator();
                if (cc==mc || (cc != null && cc.equals(mc))) {
                    map.addAllForTreeSet(set, PRESENT);
                    return true;
                }
            }
            return super.addAll(c);
        }

        // 返回子Set，实际上是通过TreeMap的subMap()实现的。
        public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                      E toElement,   boolean toInclusive) {
            return new TreeSet<E>(m.subMap(fromElement, fromInclusive,
                                           toElement,   toInclusive));
        }

        // 返回Set的头部，范围是：从头部到toElement。
        // inclusive是是否包含toElement的标志
        public NavigableSet<E> headSet(E toElement, boolean inclusive) {
            return new TreeSet<E>(m.headMap(toElement, inclusive));
        }

        // 返回Set的尾部，范围是：从fromElement到结尾。
        // inclusive是是否包含fromElement的标志
        public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
            return new TreeSet<E>(m.tailMap(fromElement, inclusive));
        }

        // 返回子Set。范围是：从fromElement(包括)到toElement(不包括)。
        public SortedSet<E> subSet(E fromElement, E toElement) {
            return subSet(fromElement, true, toElement, false);
        }

        // 返回Set的头部，范围是：从头部到toElement(不包括)。
        public SortedSet<E> headSet(E toElement) {
            return headSet(toElement, false);
        }

        // 返回Set的尾部，范围是：从fromElement到结尾(不包括)。
        public SortedSet<E> tailSet(E fromElement) {
            return tailSet(fromElement, true);
        }

        // 返回Set的比较器
        public Comparator<? super E> comparator() {
            return m.comparator();
        }

        // 返回Set的第一个元素
        public E first() {
            return m.firstKey();
        }

        // 返回Set的最后一个元素
        public E first() {
        public E last() {
            return m.lastKey();
        }

        // 返回Set中小于e的最大元素
        public E lower(E e) {
            return m.lowerKey(e);
        }

        // 返回Set中小于/等于e的最大元素
        public E floor(E e) {
            return m.floorKey(e);
        }

        // 返回Set中大于/等于e的最小元素
        public E ceiling(E e) {
            return m.ceilingKey(e);
        }

        // 返回Set中大于e的最小元素
        public E higher(E e) {
            return m.higherKey(e);
        }

        // 获取第一个元素，并将该元素从TreeMap中删除。
        public E pollFirst() {
            Map.Entry<E,?> e = m.pollFirstEntry();
            return (e == null)? null : e.getKey();
        }

        // 获取最后一个元素，并将该元素从TreeMap中删除。
        public E pollLast() {
            Map.Entry<E,?> e = m.pollLastEntry();
            return (e == null)? null : e.getKey();
        }

        // 克隆一个TreeSet，并返回Object对象
        public Object clone() {
            TreeSet<E> clone = null;
            try {
                clone = (TreeSet<E>) super.clone();
            } catch (CloneNotSupportedException e) {
                throw new InternalError();
            }

            clone.m = new TreeMap<E,Object>(m);
            return clone;
        }

        // java.io.Serializable的写入函数
        // 将TreeSet的“比较器、容量，所有的元素值”都写入到输出流中
        private void writeObject(java.io.ObjectOutputStream s)
            throws java.io.IOException {
            s.defaultWriteObject();

            // 写入比较器
            s.writeObject(m.comparator());

            // 写入容量
            s.writeInt(m.size());

            // 写入“TreeSet中的每一个元素”
            for (Iterator i=m.keySet().iterator(); i.hasNext(); )
                s.writeObject(i.next());
        }

        // java.io.Serializable的读取函数：根据写入方式读出
        // 先将TreeSet的“比较器、容量、所有的元素值”依次读出
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            // Read in any hidden stuff
            s.defaultReadObject();

            // 从输入流中读取TreeSet的“比较器”
            Comparator<? super E> c = (Comparator<? super E>) s.readObject();

            TreeMap<E,Object> tm;
            if (c==null)
                tm = new TreeMap<E,Object>();
            else
                tm = new TreeMap<E,Object>(c);
            m = tm;

            // 从输入流中读取TreeSet的“容量”
            int size = s.readInt();

            // 从输入流中读取TreeSet的“全部元素”
            tm.readTreeSet(size, s, PRESENT);
        }

        // TreeSet的序列版本号
        private static final long serialVersionUID = -2479143000061671589L;
    }

总结：

(01) TreeSet实际上是TreeMap实现的。当我们构造TreeSet时；若使用不带参数的构造函数，则TreeSet的使用自然比较器；若用户需要使用自定义的比较器，则需要使用带比较器的参数。  
(02) TreeSet是非线程安全的。  
(03) TreeSet实现java.io.Serializable的方式。当写入到输出流时，依次写入“比较器、容量、全部元素”；当读出输入流时，再依次读取。

 
 
<a name="anchor4"></a>
# 第4部分 TreeSet遍历方式

## 4.1 Iterator顺序遍历

    for(Iterator iter = set.iterator(); iter.hasNext(); ) { 
        iter.next();
    }   

## 4.2 Iterator顺序遍历

    // 假设set是TreeSet对象
    for(Iterator iter = set.descendingIterator(); iter.hasNext(); ) { 
        iter.next();
    }

## 4.3 for-each遍历HashSet

    // 假设set是TreeSet对象，并且set中元素是String类型
    String[] arr = (String[])set.toArray(new String[0]);
    for (String str:arr)
        System.out.printf("for each : %s\n", str);


TreeSet不支持快速随机遍历，只能通过迭代器进行遍历！

 

TreeSet遍历测试程序如下：

    import java.util.*;

    /**
     * @desc TreeSet的遍历程序
     *
     * @author skywang
     * @email kuiwu-wang@163.com
     */
    public class TreeSetIteratorTest {

        public static void main(String[] args) {
            TreeSet set = new TreeSet();
            set.add("aaa");
            set.add("aaa");
            set.add("bbb");
            set.add("eee");
            set.add("ddd");
            set.add("ccc");

            // 顺序遍历TreeSet
            ascIteratorThroughIterator(set) ;
            // 逆序遍历TreeSet
            descIteratorThroughIterator(set);
            // 通过for-each遍历TreeSet。不推荐！此方法需要先将Set转换为数组
            foreachTreeSet(set);
        }

        // 顺序遍历TreeSet
        public static void ascIteratorThroughIterator(TreeSet set) {
            System.out.print("\n ---- Ascend Iterator ----\n");
            for(Iterator iter = set.iterator(); iter.hasNext(); ) {
                System.out.printf("asc : %s\n", iter.next());
            }
        }

        // 逆序遍历TreeSet
        public static void descIteratorThroughIterator(TreeSet set) {
            System.out.printf("\n ---- Descend Iterator ----\n");
            for(Iterator iter = set.descendingIterator(); iter.hasNext(); )
                System.out.printf("desc : %s\n", (String)iter.next());
        }

        // 通过for-each遍历TreeSet。不推荐！此方法需要先将Set转换为数组
        private static void foreachTreeSet(TreeSet set) {
            System.out.printf("\n ---- For-each ----\n");
            String[] arr = (String[])set.toArray(new String[0]);
            for (String str:arr)
                System.out.printf("for each : %s\n", str);
        }
    }

运行结果：

     ---- Ascend Iterator ----
    asc : aaa
    asc : bbb
    asc : ccc
    asc : ddd
    asc : eee

     ---- Descend Iterator ----
    desc : eee
    desc : ddd
    desc : ccc
    desc : bbb
    desc : aaa

     ---- For-each ----
    for each : aaa
    for each : bbb
    for each : ccc
    for each : ddd
    for each : eee

 
 
<a name="anchor5"></a>
# 第5部分 TreeSet示例

下面通过实例学习如何使用TreeSet

    import java.util.*;

    /**
     * @desc TreeSet的API测试
     *
     * @author skywang
     * @email kuiwu-wang@163.com
     */
    public class TreeSetTest {

        public static void main(String[] args) {
            testTreeSetAPIs();
        }
        
        // 测试TreeSet的api
        public static void testTreeSetAPIs() {
            String val;

            // 新建TreeSet
            TreeSet tSet = new TreeSet();
            // 将元素添加到TreeSet中
            tSet.add("aaa");
            // Set中不允许重复元素，所以只会保存一个“aaa”
            tSet.add("aaa");
            tSet.add("bbb");
            tSet.add("eee");
            tSet.add("ddd");
            tSet.add("ccc");
            System.out.println("TreeSet:"+tSet);

            // 打印TreeSet的实际大小
            System.out.printf("size : %d\n", tSet.size());

            // 导航方法
            // floor(小于、等于)
            System.out.printf("floor bbb: %s\n", tSet.floor("bbb"));
            // lower(小于)
            System.out.printf("lower bbb: %s\n", tSet.lower("bbb"));
            // ceiling(大于、等于)
            System.out.printf("ceiling bbb: %s\n", tSet.ceiling("bbb"));
            System.out.printf("ceiling eee: %s\n", tSet.ceiling("eee"));
            // ceiling(大于)
            System.out.printf("higher bbb: %s\n", tSet.higher("bbb"));
            // subSet()
            System.out.printf("subSet(aaa, true, ccc, true): %s\n", tSet.subSet("aaa", true, "ccc", true));
            System.out.printf("subSet(aaa, true, ccc, false): %s\n", tSet.subSet("aaa", true, "ccc", false));
            System.out.printf("subSet(aaa, false, ccc, true): %s\n", tSet.subSet("aaa", false, "ccc", true));
            System.out.printf("subSet(aaa, false, ccc, false): %s\n", tSet.subSet("aaa", false, "ccc", false));
            // headSet()
            System.out.printf("headSet(ccc, true): %s\n", tSet.headSet("ccc", true));
            System.out.printf("headSet(ccc, false): %s\n", tSet.headSet("ccc", false));
            // tailSet()
            System.out.printf("tailSet(ccc, true): %s\n", tSet.tailSet("ccc", true));
            System.out.printf("tailSet(ccc, false): %s\n", tSet.tailSet("ccc", false));


            // 删除“ccc”
            tSet.remove("ccc");
            // 将Set转换为数组
            String[] arr = (String[])tSet.toArray(new String[0]);
            for (String str:arr)
                System.out.printf("for each : %s\n", str);

            // 打印TreeSet
            System.out.printf("TreeSet:%s\n", tSet);

            // 遍历TreeSet
            for(Iterator iter = tSet.iterator(); iter.hasNext(); ) {
                System.out.printf("iter : %s\n", iter.next());
            }

            // 删除并返回第一个元素
            val = (String)tSet.pollFirst();
            System.out.printf("pollFirst=%s, set=%s\n", val, tSet);

            // 删除并返回最后一个元素
            val = (String)tSet.pollLast();
            System.out.printf("pollLast=%s, set=%s\n", val, tSet);

            // 清空HashSet
            tSet.clear();

            // 输出HashSet是否为空
            System.out.printf("%s\n", tSet.isEmpty()?"set is empty":"set is not empty");
        }
    }

运行结果： 

    TreeSet:[aaa, bbb, ccc, ddd, eee]
    size : 5
    floor bbb: bbb
    lower bbb: aaa
    ceiling bbb: bbb
    ceiling eee: eee
    higher bbb: ccc
    subSet(aaa, true, ccc, true): [aaa, bbb, ccc]
    subSet(aaa, true, ccc, false): [aaa, bbb]
    subSet(aaa, false, ccc, true): [bbb, ccc]
    subSet(aaa, false, ccc, false): [bbb]
    headSet(ccc, true): [aaa, bbb, ccc]
    headSet(ccc, false): [aaa, bbb]
    tailSet(ccc, true): [ccc, ddd, eee]
    tailSet(ccc, false): [ddd, eee]
    for each : aaa
    for each : bbb
    for each : ddd
    for each : eee
    TreeSet:[aaa, bbb, ddd, eee]
    iter : aaa
    iter : bbb
    iter : ddd
    iter : eee
    pollFirst=aaa, set=[bbb, ddd, eee]
    pollLast=eee, set=[bbb, ddd]
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
