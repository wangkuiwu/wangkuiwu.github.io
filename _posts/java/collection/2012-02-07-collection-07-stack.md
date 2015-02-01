---
layout: post
title: "Java 集合系列07之 Stack详细介绍(源码解析)和使用示例"
description: "java collection"
category: java
tags: [java]
date: 2012-02-07 09:01
---


 
> 学完Vector了之后，接下来我们开始学习Stack。Stack很简单，它继承于Vector。学习方式还是和之前一样，先对Stack有个整体认识，然后再学习它的源码；最后再通过实例来学会使用它。内容包括：

> **目录**  
> [第1部分 Stack介绍](#anchor1)   
> [第2部分 Stack源码解析(基于JDK1.6.0_45)](#anchor2)   
> [第3部分 Vector示例](#anchor3)   

 
<a name="anchor1"></a>
# 第1部分 Stack介绍

## Stack简介

Stack是栈。它的特性是：先进后出(FILO, First In Last Out)。

java工具包中的Stack是继承于Vector(矢量队列)的，由于Vector是通过数组实现的，这就意味着，Stack也是通过数组实现的，而非链表。当然，我们也可以将LinkedList当作栈来使用！在“Java 集合系列06之 Vector详细介绍(源码解析)和使用示例”中，已经详细介绍过Vector的数据结构，这里就不再对Stack的数据结构进行说明了。

 

Stack的继承关系

    java.lang.Object
    ↳     java.util.AbstractCollection<E>
       ↳     java.util.AbstractList<E>
           ↳     java.util.Vector<E>
               ↳     java.util.Stack<E>

Stack的声明

    public class Stack<E> extends Vector<E> {}

 

Stack和Collection的关系如下图：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/pictures/java/collections/collection07.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/pictures/java/collections/collection07.jpg" alt="" /></a>
 

Stack的构造函数

    Stack()
 

Stack的API

             boolean       empty()
    synchronized E             peek()
    synchronized E             pop()
                 E             push(E object)
    synchronized int           search(Object o)

 

由于Stack和继承于Vector，因此它也包含Vector中的全部API。

 
<a name="anchor2"></a>
# 第2部分 Stack源码解析(基于JDK1.6.0_45)

Stack的源码非常简单，下面我们对它进行学习。 

    package java.util;

    public
    class Stack<E> extends Vector<E> {
        // 版本ID。这个用于版本升级控制，这里不须理会！
        private static final long serialVersionUID = 1224463164541339165L;

        // 构造函数
        public Stack() {
        }

        // push函数：将元素存入栈顶
        public E push(E item) {
            // 将元素存入栈顶。
            // addElement()的实现在Vector.java中
            addElement(item);

            return item;
        }

        // pop函数：返回栈顶元素，并将其从栈中删除
        public synchronized E pop() {
            E    obj;
            int    len = size();

            obj = peek();
            // 删除栈顶元素，removeElementAt()的实现在Vector.java中
            removeElementAt(len - 1);

            return obj;
        }

        // peek函数：返回栈顶元素，不执行删除操作
        public synchronized E peek() {
            int    len = size();

            if (len == 0)
                throw new EmptyStackException();
            // 返回栈顶元素，elementAt()具体实现在Vector.java中
            return elementAt(len - 1);
        }

        // 栈是否为空
        public boolean empty() {
            return size() == 0;
        }

        // 查找“元素o”在栈中的位置：由栈底向栈顶方向数
        public synchronized int search(Object o) {
            // 获取元素索引，elementAt()具体实现在Vector.java中
            int i = lastIndexOf(o);

            if (i >= 0) {
                return size() - i;
            }
            return -1;
        }
    }

总结：

(01) Stack实际上也是通过数组去实现的。  
> 执行push时(即，将元素推入栈中)，是通过将元素追加的数组的末尾中。  
> 执行peek时(即，取出栈顶元素，不执行删除)，是返回数组末尾的元素。  
> 执行pull时(即，取出栈顶元素，并将该元素从栈中删除)，是取出数组末尾的元素，然后将该元素从数组中删除。  
(02) Stack继承于Vector，意味着Vector拥有的属性和功能，Stack都拥有。  


<a name="anchor3"></a>
# 第3部分 Vector示例

下面我们通过实例学习如何使用Stack

    import java.util.Stack;
    import java.util.Iterator;
    import java.util.List;

    /**
     * @desc Stack的测试程序。测试常用API的用法
     *
     * @author skywang
     */
    public class StackTest {

        public static void main(String[] args) {
            Stack stack = new Stack();
            // 将1,2,3,4,5添加到栈中
            for(int i=1; i<6; i++) {
                stack.push(String.valueOf(i));
            }

            // 遍历并打印出该栈
            iteratorThroughRandomAccess(stack) ;

            // 查找“2”在栈中的位置，并输出
            int pos = stack.search("2");
            System.out.println("the postion of 2 is:"+pos);

            // pup栈顶元素之后，遍历栈
            stack.pop();
            iteratorThroughRandomAccess(stack) ;

            // peek栈顶元素之后，遍历栈
            String val = (String)stack.peek();
            System.out.println("peek:"+val);
            iteratorThroughRandomAccess(stack) ;

            // 通过Iterator去遍历Stack
            iteratorThroughIterator(stack) ;
        }

        /**
         * 通过快速访问遍历Stack
         */
        public static void iteratorThroughRandomAccess(List list) {
            String val = null;
            for (int i=0; i<list.size(); i++) {
                val = (String)list.get(i);
                System.out.print(val+" ");
            }
            System.out.println();
        }

        /**
         * 通过迭代器遍历Stack
         */
        public static void iteratorThroughIterator(List list) {

            String val = null;
            for(Iterator iter = list.iterator(); iter.hasNext(); ) {
                val = (String)iter.next();
                System.out.print(val+" ");
            }
            System.out.println();
        }

    }

运行结果： 

    1 2 3 4 5 
    the postion of 2 is:4
    1 2 3 4 
    peek:4
    1 2 3 4 
    1 2 3 4 


