---
layout: post
title: "基数排序(C/C++/Java)"
description: "radix sort"
category: datastructure
tags: [algrithm,sort,c,c++,java]
date: 2013-05-10 09:27
---



> 本章会对排序算法中的基数排序进行图文详解，并给出C/C++/Java的实现。



# 基数排序介绍

基数排序(Radix Sort)是桶排序的扩展，它的基本思想是：将整数按位数切割成不同的数字，然后按每个位数分别比较。

具体做法是：将所有待比较数值统一为同样的数位长度，数位较短的数前面补零。然后，从最低位开始，依次进行一次排序。这样从最低位排序一直到最高位排序完成以后, 数列就变成一个有序序列。




# 基数排序图文说明

## 1. 基数排序图解

通过基数排序对数组{53, 3, 542, 748, 14, 214, 154, 63, 616}，它的示意图如下：

![img](/media/pic/datastruct_algrithm/algrithm/radix_01.jpg)

在上图中，首先将所有待比较树脂统一为统一位数长度，接着从最低位开始，依次进行排序。

> 1. 按照个位数进行排序。
> 2. 按照十位数进行排序。
> 3. 按照百位数进行排序。

排序后，数列就变成了一个有序序列。


## 2. 基数排序代码

### 2.1 基数排序

    /*
     * 基数排序
     *
     * 参数说明：
     *     a -- 数组
     *     n -- 数组长度
     */
    void radix_sort(int a[], int n)
    {
        int exp;	// 指数。当对数组按各位进行排序时，exp=1；按十位进行排序时，exp=10；...
        int max = get_max(a, n);	// 数组a中的最大值

        // 从个位开始，对数组a按"指数"进行排序
        for (exp = 1; max/exp > 0; exp *= 10)
            count_sort(a, n, exp);
    }

radix_sort(a, n)的作用是对数组a进行排序。

> + 首先通过get_max(a)获取数组a中的最大值。获取最大值的目的是计算出数组a的最大指数。  
> + 获取到数组a中的最大指数之后，再从指数1开始，根据位数对数组a中的元素进行排序。排序的时候采用了桶排序。  
> + count_sort(a, n, exp)的作用是对数组a按照指数exp进行排序。 

### 2.2 获取最大值 

    /* 
     * 获取数组a中最大值
     *
     * 参数说明：
     *     a -- 数组
     *     n -- 数组长度
     */
    int get_max(int a[], int n)
    {
        int i, max;

        max = a[0];
        for (i = 1; i < n; i++)
            if (a[i] > max)
                max = a[i];
        return max;
    }

### 2.3 指数排序

    /*
     * 对数组按照"某个位数"进行排序(桶排序)
     *
     * 参数说明：
     *     a -- 数组
     *     n -- 数组长度
     *     exp -- 指数。对数组a按照该指数进行排序。
     *
     * 例如，对于数组a={50, 3, 542, 745, 2014, 154, 63, 616}；
     *    (01) 当exp=1表示按照"个位"对数组a进行排序
     *    (02) 当exp=10表示按照"十位"对数组a进行排序
     *    (03) 当exp=100表示按照"百位"对数组a进行排序
     *    ...
     */
    void count_sort(int a[], int n, int exp)
    {
        int output[n]; 			// 存储"被排序数据"的临时数组
        int i, buckets[10] = {0};

        // 将数据出现的次数存储在buckets[]中
        for (i = 0; i < n; i++)
            buckets[ (a[i]/exp)%10 ]++;

        // 更改buckets[i]。目的是让更改后的buckets[i]的值，是该数据在output[]中的位置。
        for (i = 1; i < 10; i++)
            buckets[i] += buckets[i - 1];

        // 将数据存储到临时数组output[]中
        for (i = n - 1; i >= 0; i--)
        {
            output[buckets[ (a[i]/exp)%10 ] - 1] = a[i];
            buckets[ (a[i]/exp)%10 ]--;
        }

        // 将排序好的数据赋值给a[]
        for (i = 0; i < n; i++)
            a[i] = output[i];
    }


## 3. 排序演示

下面简单介绍一下对数组{53, 3, 542, 748, 14, 214, 154, 63, 616}按个位数进行排序的流程。

(01) 个位的数值范围是[0,10)。因此，创建桶数组buckets[]，将数组按照个位数值添加到桶中。

![img](/media/pic/datastruct_algrithm/algrithm/radix_02.jpg)

(02) 接着是根据桶数组buckets[]来进行排序。假设将排序后的数组存在output[]中；找出output[]和buckets[]之间的联系就可以对数据进行排序了。

![img](/media/pic/datastruct_algrithm/algrithm/radix_03.jpg)



# 基数排序实现
1. [基数排序C实现][link_radixsort_c] (radix_sort.c)

2. [基数排序C++实现][link_radixsort_cplus] (RadixSort.cpp)

3. [基数排序Java实现][link_radixsort_java] (RadixSort.java)




[link_radixsort_c]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/radix_sort/c/radix_sort.c
[link_radixsort_cplus]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/radix_sort/cplus/RadixSort.cpp
[link_radixsort_java]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/radix_sort/java/RadixSort.java
