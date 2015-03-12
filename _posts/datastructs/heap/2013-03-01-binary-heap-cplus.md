---
layout: post
title: "二叉堆(二)之 C++详解"
description: "binary heap"
category: datastructure
tags: [heap,c++]
date: 2013-03-01 12:30
---


> [上一章][link_binaryheap_c]介绍了二叉堆的基本概念，并通过C语言实现了二叉堆。本章是二叉堆的C++实现。



# 堆和二叉堆的介绍

## 1. 堆的定义

堆(heap)，这里所说的堆是数据结构中的堆，而不是内存模型中的堆。堆通常是一个可以被看做一棵树，它满足下列性质：

+ [性质一] 堆中某个节点的值总是不大于或不小于其父节点的值；  
+ [性质二] 堆总是一棵完全树。

将根节点最大的堆叫做最大堆或大根堆，根节点最小的堆叫做最小堆或小根堆。常见的堆有二叉堆、左倾堆、斜堆、斐波那契堆等等。


## 2. 二叉堆的定义

二叉堆是完全二元树或者是近似完全二元树，它分为两种：**最大堆**和**最小堆**。示意图如下：

> 最大堆：父结点的键值总是大于或等于任何一个子节点的键值；  
> 最小堆：父结点的键值总是小于或等于任何一个子节点的键值。


![img](/media/pic/datastruct_algrithm/heap/erchadui/heap_01.jpg)

<br/>
二叉堆一般都通过"数组"来实现。数组实现的二叉堆，父节点和子节点的位置存在一定的关系。有时候，我们将"二叉堆的第一个元素"放在数组索引0的位置，有时候放在1的位置。当然，它们的本质一样(都是二叉堆)，知识实现上稍微有一丁点区别。

假设"第一个元素"在数组中的索引为 0 的话，则父节点和子节点的位置关系如下：

> (01) 索引为i的左孩子的索引是 (2\*i+1);  
> (02) 索引为i的左孩子的索引是 (2\*i+2);  
> (03) 索引为i的父结点的索引是 floor((i-1)/2)。  

![img](/media/pic/datastruct_algrithm/heap/erchadui/heap_02.jpg)


<br>
假设"第一个元素"在数组中的索引为 1 的话，则父节点和子节点的位置关系如下：

> (01) 索引为i的左孩子的索引是 (2\*i);  
> (02) 索引为i的左孩子的索引是 (2\*i+1);  
> (03) 索引为i的父结点的索引是 floor(i/2)。  

![img](/media/pic/datastruct_algrithm/heap/erchadui/heap_03.jpg)


**注意：本文二叉堆的实现统统都是采用"二叉堆第一个元素在数组索引为0"的方式！**




# 二叉堆的图文解析

本文是以"最大堆"来进行介绍的。

## 1. 基本定义

    template <class T>
    class MaxHeap{
        private:
            T *mHeap;		// 数据
            int mCapacity;	// 总的容量
            int mSize;		// 实际容量

        private:
            // 最大堆的向下调整算法
            void filterdown(int start, int end);
            // 最大堆的向上调整算法(从start开始向上直到0，调整堆)
            void filterup(int start);
        public:
            MaxHeap();
            MaxHeap(int capacity);
            ~MaxHeap();

            // 返回data在二叉堆中的索引
            int getIndex(T data);
            // 删除最大堆中的data
            int remove(T data);
            // 将data插入到二叉堆中
            int insert(T data);
            // 打印二叉堆
            void print();
    };


MaxHeap是最大堆的对应的类。

它包括的核心内容是"添加"和"删除"，理解这两个算法，二叉堆也就基本掌握了。下面对它们进行介绍。



## 2. 添加

假设在最大堆[90,80,70,60,40,30,20,10,50]种添加85，需要执行的步骤如下：

![img](/media/pic/datastruct_algrithm/heap/erchadui/heap_04.jpg)


如上图所示，当向最大堆中添加数据时：先将数据加入到最大堆的最后，然后尽可能把这个元素往上挪，直到挪不动为止！

将85添加到[90,80,70,60,40,30,20,10,50]中后，最大堆变成了[90,85,70,60,80,30,20,10,50,40]。

<br/>
**最大堆的插入代码**

    /*
     * 最大堆的向上调整算法(从start开始向上直到0，调整堆)
     *
     * 注：数组实现的堆中，第N个节点的左孩子的索引值是(2N+1)，右孩子的索引是(2N+2)。
     *
     * 参数说明：
     *     start -- 被上调节点的起始位置(一般为数组中最后一个元素的索引)
     */
    template <class T>
    void MaxHeap<T>::filterup(int start)
    {
        int c = start;			// 当前节点(current)的位置
        int p = (c-1)/2;		// 父(parent)结点的位置 
        T tmp = mHeap[c];		// 当前节点(current)的大小

        while(c > 0)
        {
            if(mHeap[p] >= tmp)
                break;
            else
            {
                mHeap[c] = mHeap[p];
                c = p;
                p = (p-1)/2;   
            }       
        }
        mHeap[c] = tmp;
    }

    /* 
     * 将data插入到二叉堆中
     *
     * 返回值：
     *     0，表示成功
     *    -1，表示失败
     */
    template <class T>
    int MaxHeap<T>::insert(T data)
    {
        // 如果"堆"已满，则返回
        if(mSize == mCapacity)
            return -1;
     
        mHeap[mSize] = data;		// 将"数组"插在表尾
        filterup(mSize);	        // 向上调整堆
        mSize++;					// 堆的实际容量+1

        return 0;
    }

insert(data)的作用：将数据data添加到最大堆中。当堆已满的时候，添加失败；否则data添加到最大堆的末尾。然后通过上调算法重新调整数组，使之重新成为最大堆。


## 3. 删除

假设从最大堆[90,85,70,60,80,30,20,10,50,40]中删除90，需要执行的步骤如下：

![img](/media/pic/datastruct_algrithm/heap/erchadui/heap_05.jpg)


从[90,85,70,60,80,30,20,10,50,40]删除90之后，最大堆变成了[85,80,70,60,40,30,20,10,50]。

如上图所示，当从最大堆中删除数据时：先删除该数据，然后用最大堆中最后一个的元素插入这个空位；接着，把这个“空位”尽量往上挪，直到剩余的数据变成一个最大堆。

注意：考虑从最大堆[90,85,70,60,80,30,20,10,50,40]中删除70，执行的步骤不能单纯的用它的字节点来替换；而必须考虑到"替换后的树仍然要是最大堆"！

![img](/media/pic/datastruct_algrithm/heap/erchadui/heap_06.jpg)


**最大堆的删除代码**

    /* 
     * 最大堆的向下调整算法
     *
     * 注：数组实现的堆中，第N个节点的左孩子的索引值是(2N+1)，右孩子的索引是(2N+2)。
     *
     * 参数说明：
     *     start -- 被下调节点的起始位置(一般为0，表示从第1个开始)
     *     end   -- 截至范围(一般为数组中最后一个元素的索引)
     */
    template <class T>
    void MaxHeap<T>::filterdown(int start, int end)
    {
        int c = start; 	 	// 当前(current)节点的位置
        int l = 2*c + 1; 	// 左(left)孩子的位置
        T tmp = mHeap[c];	// 当前(current)节点的大小

        while(l <= end)
        {
            // "l"是左孩子，"l+1"是右孩子
            if(l < end && mHeap[l] < mHeap[l+1])
                l++;		// 左右两孩子中选择较大者，即mHeap[l+1]
            if(tmp >= mHeap[l])
                break;		//调整结束
            else
            {
                mHeap[c] = mHeap[l];
                c = l;
                l = 2*l + 1;   
            }       
        }   
        mHeap[c] = tmp;
    }

    /*
     * 删除最大堆中的data
     *
     * 返回值：
     *      0，成功
     *     -1，失败
     */
    template <class T>
    int MaxHeap<T>::remove(T data)
    {
        int index;
        // 如果"堆"已空，则返回-1
        if(mSize == 0)
            return -1;

        // 获取data在数组中的索引
        index = getIndex(data); 
        if (index==-1)
            return -1;

        mHeap[index] = mHeap[--mSize];	// 用最后元素填补
        filterdown(index, mSize-1);		// 从index位置开始自上向下调整为最大堆

        return 0;
    }


# 二叉堆的实现源码和测试包括

二叉堆的源码包含了"最大堆"和"最小堆"。

1. [最大堆(MaxHeap.cpp)][link_maxheap_cpp] 

2. [最小堆(MinHeap.cpp)][link_minheap_cpp] 


PS. 二叉堆是"[堆排序][link_heapsort]"的理论基石。后面的算法中会讲解到"[堆排序][link_heapsort]"，理解了"二叉堆"之后，"[堆排序][link_heapsort]"就很简单了。


[link_maxheap_cpp]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/two_cha/cplus/MaxHeap.cpp
[link_minheap_cpp]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/two_cha/cplus/MinHeap.cpp
[link_heapsort]: /2013/05/06/heap-sort/
[link_binaryheap_c]: /2013/03/01/binary-heap-c/

