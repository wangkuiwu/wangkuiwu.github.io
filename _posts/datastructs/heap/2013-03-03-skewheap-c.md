---
layout: post
title: "斜堆(一)之 C语言详解"
description: "skew heap"
category: datastructure
tags: [heap,c]
date: 2013-03-03 09:11
---


> 本章介绍斜堆。和以往一样，本文会先对斜堆的理论知识进行简单介绍，然后给出C语言的实现。后续再分别给出C++和Java版本的实现；实现的语言虽不同，但是原理如出一辙，选择其中之一进行了解即可。若文章有错误或不足的地方，请不吝指出！ 



# 斜堆的介绍

斜堆(Skew heap)也叫自适应堆(self-adjusting heap)，它是左倾堆的一个变种。它的合并操作的时间复杂度也是O(log n)。

相比于[左倾堆][link_leftist_c]，斜堆的节点没有"零距离"这个属性。除此之外，它们斜堆的合并操作也不同。斜堆的合并操作算法如下：

> 1. 如果一个空斜堆与一个非空斜堆合并，返回非空斜堆。  
> 2. 如果两个斜堆都非空，那么比较两个根节点，取较小堆的根节点为新的根节点。将"较小堆的根节点的右孩子"和"较大堆"进行合并。  
> 3. 合并后，交换新堆根节点的左孩子和右孩子。  

第3步是斜堆和左倾堆的合并操作差别的关键所在，如果是左倾堆，则合并后要比较左右孩子的零距离大小，若右孩子的零距离 > 左孩子的零距离，则交换左右孩子；最后，在设置根的零距离。


# 斜堆的基本操作

## 1. 结构体

    typedef int Type;

    typedef struct _SkewNode{
        Type   key;					// 关键字(键值)
        struct _SkewNode *left;	// 左孩子
        struct _SkewNode *right;	// 右孩子
    }SkewNode, *SkewHeap;

SkewNode是斜堆对应的节点类。


## 2. 合并

    /* 
     * 合并"斜堆x"和"斜堆y"
     *
     * 返回值：
     *     合并得到的树的根节点
     */
    SkewNode* merge_skewheap(SkewHeap x, SkewHeap y)
    {
        if(x == NULL)
            return y;
        if(y == NULL)
            return x;

        // 合并x和y时，将x作为合并后的树的根；
        // 这里的操作是保证: x的key < y的key
        if(x->key > y->key)
            swap_skewheap_node(x, y);

        // 将x的右孩子和y合并，
        // 合并后直接交换x的左右孩子，而不需要像左倾堆一样考虑它们的npl。
        SkewNode *tmp = merge_skewheap(x->right, y);
        x->right = x->left;
        x->left  = tmp;

        return x;
    }

merge_skewheap(x, y)的作用是合并x和y这两个斜堆，并返回得到的新堆。merge_skewheap(x, y)是递归实现的。

## 3. 添加

    /* 
     * 新建结点(key)，并将其插入到斜堆中
     *
     * 参数说明：
     *     heap 斜堆的根结点
     *     key 插入结点的键值
     * 返回值：
     *     根节点
     */
    SkewNode* insert_skewheap(SkewHeap heap, Type key)
    {
        SkewNode *node;	// 新建结点

        // 如果新建结点失败，则返回。
        if ((node = (SkewNode *)malloc(sizeof(SkewNode))) == NULL)
            return heap;
        node->key = key;
        node->left = node->right = NULL;

        return merge_skewheap(heap, node);
    }

insert_skewheap(heap, key)的作用是新建键值为key的结点，并将其插入到斜堆中，并返回堆的根节点。

## 4. 删除

    /* 
     * 取出根节点
     *
     * 返回值：
     *     取出根节点后的新树的根节点
     */
    SkewNode* delete_skewheap(SkewHeap heap)
    {
        SkewNode *l = heap->left;
        SkewNode *r = heap->right;

        // 删除根节点
        free(heap);

        return merge_skewheap(l, r); // 返回左右子树合并后的新树
    }

delete_skewheap(heap)的作用是删除斜堆的最小节点，并返回删除节点后的斜堆根节点。

<br/>
*PS. 注意：关于斜堆的"前序遍历"、"中序遍历"、"后序遍历"、"打印"、"销毁"等接口就不再单独介绍了。后文的源码中有给出它们的实现代码，Please RTFSC(Read The Fucking Source Code)！*




# 斜堆的完整源码

包含测试程序在内，一共3个文件。

1. [斜堆的头文件(skewheap.h)][link_skewheap_c_01] 

2. [斜堆的实现文件(skewheap.c)][link_skewheap_c_02] 

3. [斜堆的测试程序(skewheap_test.c)][link_skewheap_c_03] 



[link_skewheap_c_01]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/c/skewheap.h
[link_skewheap_c_02]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/c/skewheap.c
[link_skewheap_c_03]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/c/skewheap_test.c
[link_leftist_c]: /2013/03/02/leftist-c/

