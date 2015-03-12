---
layout: post
title: "选择排序(C/C++/Java)"
description: "select sort"
category: datastructure
tags: [algrithm,sort,c,c++,java]
date: 2013-05-05 09:28
---



> 本章会对排序算法中的选择排序进行图文详解，并给出C/C++/Java的实现。



# 选择排序介绍

选择排序(Selection sort)是一种简单直观的排序算法。

它的基本思想是：首先在未排序的数列中找到最小(or最大)元素，然后将其存放到数列的起始位置；接着，再从剩余未排序的元素中继续寻找最小(or最大)元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。



# 选择排序图文说明

选择排序代码

    /*
     * 选择排序
     *
     * 参数说明：
     *     a -- 待排序的数组
     *     n -- 数组的长度
     */
    void select_sort(int a[], int n)
    {
        int i;		// 有序区的末尾位置
        int j;		// 无序区的起始位置
        int min;	// 无序区中最小元素位置

        for(i=0; i<n; i++)
        {
            min=i;

            // 找出"a[i+1] ... a[n]"之间的最小元素，并赋值给min。
            for(j=i+1; j<n; j++)
            {
                if(a[j] < a[min])
                    min=j;
            }

            // 若min!=i，则交换 a[i] 和 a[min]。
            // 交换之后，保证了a[0] ... a[i] 之间的元素是有序的。
            if(min != i)
                swap(a[i], a[min]);
        }
    }


下面以数列{20,40,30,10,60,50}为例，演示它的选择排序过程。

![img](/media/pic/datastruct_algrithm/algrithm/select_01.jpg)


**排序流程**

+ 第1趟：i=0。找出a[1...5]中的最小值a[3]=10，然后将a[0]和a[3]互换。 数列变化：20,40,30,10,60,50  -- >  10,40,30,20,60,50
+ 第2趟：i=1。找出a[2...5]中的最小值a[3]=20，然后将a[1]和a[3]互换。 数列变化：10,40,30,20,60,50  -- >  10,20,30,40,60,50
+ 第3趟：i=2。找出a[3...5]中的最小值，由于该最小值大于a[2]，该趟不做任何处理。 
+ 第4趟：i=3。找出a[4...5]中的最小值，由于该最小值大于a[3]，该趟不做任何处理。 
+ 第5趟：i=4。交换a[4]和a[5]的数据。 数列变化：10,20,30,40,60,50  -- >  10,20,30,40,50,60




# 选择排序的时间复杂度和稳定性

**选择排序时间复杂度**

选择排序的时间复杂度是O(N<sup>2</sup>)。

假设被排序的数列中有N个数。遍历一趟的时间复杂度是O(N)，需要遍历多少次呢？N-1！因此，选择排序的时间复杂度是O(N<sup>2</sup>)。

<br/>
**选择排序稳定性**

选择排序是稳定的算法，它满足稳定算法的定义。

*算法稳定性 -- 假设在数列中存在a[i]=a[j]，若在排序之前，a[i]在a[j]前面；并且排序之后，a[i]仍然在a[j]前面。则这个排序算法是稳定的！*



# 选择排序实现

1. [选择排序C实现][link_selectsort_c] (select_sort.c)
2. [选择排序C++实现][link_selectsort_cplus] (SelectSort.cpp)
3. [选择排序Java实现][link_selectsort_java] (SelectSort.java)


[link_selectsort_c]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/selection_sort/c/select_sort.c
[link_selectsort_cplus]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/selection_sort/cplus/SelectSort.cpp
[link_selectsort_java]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/selection_sort/java/SelectSort.java
