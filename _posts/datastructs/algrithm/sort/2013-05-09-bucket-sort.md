---
layout: post
title: "桶排序(C/C++/Java)"
description: "bucket sort"
category: datastructure
tags: [algrithm,sort,c,c++,java]
date: 2013-05-09 09:28
---



> 本章会对排序算法中的桶排序进行图文详解，并给出C/C++/Java的实现。



# 桶排序介绍

桶排序(Bucket Sort)的原理很简单，它是将数组分到有限数量的桶子里。

假设待排序的数组a中共有N个整数，并且已知数组a中数据的范围[0, MAX)。在桶排序时，创建容量为MAX的桶数组r，并将桶数组元素都初始化为0；将容量为MAX的桶数组中的每一个单元都看作一个"桶"。

在排序时，逐个遍历数组a，将数组a的值，作为"桶数组r"的下标。当a中数据被读取时，就将桶的值加1。例如，读取到数组a[3]=5，则将r[5]的值+1。


# 桶排序图文说明

桶排序代码

    /*
     * 桶排序
     *
     * 参数说明：
     *     a -- 待排序数组
     *     n -- 数组a的长度
     *     max -- 数组a中最大值的范围
     */
    void bucket_sort(int a[], int n, int max)
    {
        int i, j;
        int *buckets;

        if (a==NULL || n<1 || max<1)
            return ;

        // 创建一个容量为max的数组buckets，并且将buckets中的所有数据都初始化为0。
        if ((buckets=(int *)malloc(max*sizeof(int)))==NULL)
            return ;
        memset(buckets, 0, max*sizeof(int));

        // 1. 计数
        for(i = 0; i < n; i++) 
            buckets[a[i]]++; 

        // 2. 排序
        for (i = 0, j = 0; i < max; i++) 
            while( (buckets[i]--) >0 )
                a[j++] = i;

        free(buckets);
    }

bucketSort(a, n, max)是作用是对数组a进行桶排序，n是数组a的长度，max是数组中最大元素所属的范围[0,max)。

假设a={8,2,3,4,3,6,6,3,9}, max=10。此时，将数组a的所有数据都放到需要为0-9的桶中。如下图：

![img](/media/pic/datastruct_algrithm/algrithm/bucket_01.jpg)

在将数据放到桶中之后，再通过一定的算法，将桶中的数据提出出来并转换成有序数组。就得到我们想要的结果了。


# 桶排序实现

1. [桶排序C实现][link_bucketsort_c] (bucket_sort.c)

2. [桶排序C++实现][link_bucketsort_cplus] (BucketSort.cpp)

3. [桶排序Java实现][link_bucketsort_java] (BucketSort.java)



[link_bucketsort_c]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/bucket_sort/c/bucket_sort.c
[link_bucketsort_cplus]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/bucket_sort/cplus/BucketSort.cpp
[link_bucketsort_java]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/algrightm/sort/bucket_sort/java/BucketSort.java
