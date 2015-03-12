---
layout: post
title: "斜堆(二)之 C++详解"
description: "skew heap"
category: datastructure
tags: [heap,c++]
date: 2013-03-03 12:11
---


> [上一章][link_skewheap_c]介绍了斜堆的基本概念，并通过C语言实现了斜堆。本章是斜堆的C++实现。



# 斜堆的介绍

斜堆(Skew heap)也叫自适应堆(self-adjusting heap)，它是左倾堆的一个变种。它的合并操作的时间复杂度也是O(log n)。

相比于[左倾堆][link_leftist_cplus]，斜堆的节点没有"零距离"这个属性。除此之外，它们斜堆的合并操作也不同。斜堆的合并操作算法如下：

> 1. 如果一个空斜堆与一个非空斜堆合并，返回非空斜堆。  
> 2. 如果两个斜堆都非空，那么比较两个根节点，取较小堆的根节点为新的根节点。将"较小堆的根节点的右孩子"和"较大堆"进行合并。  
> 3. 合并后，交换新堆根节点的左孩子和右孩子。  

第3步是斜堆和左倾堆的合并操作差别的关键所在，如果是左倾堆，则合并后要比较左右孩子的零距离大小，若右孩子的零距离 > 左孩子的零距离，则交换左右孩子；最后，在设置根的零距离。


# 斜堆的基本操作

## 1. 基本定义

    template <class T>
    class SkewNode{
        public:
            T key;				// 关键字(键值)
            SkewNode *left;		// 左孩子
            SkewNode *right;	// 右孩子

            SkewNode(T value, SkewNode *l, SkewNode *r):
                key(value), left(l),right(r) {}
    };

    template <class T>
    class SkewHeap {
        private:
            SkewNode<T> *mRoot;	// 根结点

        public:
            SkewHeap();
            ~SkewHeap();

            // 前序遍历"斜堆"
            void preOrder();
            // 中序遍历"斜堆"
            void inOrder();
            // 后序遍历"斜堆"
            void postOrder();

            // 将other的斜堆合并到this中。
            void merge(SkewHeap<T>* other);
            // 将结点(key为节点键值)插入到斜堆中
            void insert(T key);
            // 删除结点(key为节点键值)
            void remove();

            // 销毁斜堆
            void destroy();

            // 打印斜堆
            void print();
        private:

            // 前序遍历"斜堆"
            void preOrder(SkewNode<T>* heap) const;
            // 中序遍历"斜堆"
            void inOrder(SkewNode<T>* heap) const;
            // 后序遍历"斜堆"
            void postOrder(SkewNode<T>* heap) const;

            // 交换节点x和节点y
            void swapNode(SkewNode<T> *&x, SkewNode<T> *&y);
            // 合并"斜堆x"和"斜堆y"
            SkewNode<T>* merge(SkewNode<T>* &x, SkewNode<T>* &y);

            // 销毁斜堆
            void destroy(SkewNode<T>* &heap);

            // 打印斜堆
            void print(SkewNode<T>* heap, T key, int direction);
    };

SkewNode是斜堆对应的节点类。
SkewHeap是斜堆类，它包含了斜堆的根节点，以及斜堆的操作。


## 2. 合并

    /*
     * 合并"斜堆x"和"斜堆y"
     */
    template <class T>
    SkewNode<T>* SkewHeap<T>::merge(SkewNode<T>* &x, SkewNode<T>* &y)
    {
        if(x == NULL)
            return y;
        if(y == NULL)
            return x;

        // 合并x和y时，将x作为合并后的树的根；
        // 这里的操作是保证: x的key < y的key
        if(x->key > y->key)
            swapNode(x, y);

        // 将x的右孩子和y合并，
        // 合并后直接交换x的左右孩子，而不需要像左倾堆一样考虑它们的npl。
        SkewNode<T> *tmp = merge(x->right, y);
        x->right = x->left;
        x->left  = tmp;

        return x;
    }

    /*
     * 将other的斜堆合并到this中。
     */
    template <class T>
    void SkewHeap<T>::merge(SkewHeap<T>* other)
    {
        mRoot = merge(mRoot, other->mRoot);
    }

merge(x, y)是内部接口，作用是合并x和y这两个斜堆，并返回得到的新堆的根节点。

merge(other)是外部接口，作用是将other合并到当前堆中。



## 3. 添加

    /* 
     * 新建键值为key的结点并将其插入到斜堆中
     *
     * 参数说明：
     *     heap 斜堆的根结点
     *     key 插入的结点的键值
     * 返回值：
     *     根节点
     */
    template <class T>
    void SkewHeap<T>::insert(T key)
    {
        SkewNode<T> *node;	// 新建结点

        // 新建节点
        node = new SkewNode<T>(key, NULL, NULL);
        if (node==NULL)
        {
            cout << "ERROR: create node failed!" << endl;
            return ;
        }

        mRoot = merge(mRoot, node);
    }

insert(key)的作用是新建键值为key的节点，并将其加入到当前斜堆中。

## 4. 删除

    /* 
     * 删除结点
     */
    template <class T>
    void SkewHeap<T>::remove()
    {
        if (mRoot == NULL)
            return NULL;

        SkewNode<T> *l = mRoot->left;
        SkewNode<T> *r = mRoot->right;

        // 删除根节点
        delete mRoot;
        // 左右子树合并后的新树
        mRoot = merge(l, r); 
    }

remove()的作用是删除斜堆的最小节点。

<br/>
*PS. 注意：关于斜堆的"前序遍历"、"中序遍历"、"后序遍历"、"打印"、"销毁"等接口就不再单独介绍了。后文的源码中有给出它们的实现代码，Please RTFSC(Read The Fucking Source Code)！*




# 斜堆的完整源码

包含测试程序在内，一共2个文件。

1. [斜堆的实现文件(SkewHeap.h)][link_skewheap_cplus_01] 

2. [斜堆的测试程序(SkewHeapTest.cpp)][link_skewheap_cplus_02] 



[link_skewheap_cplus_01]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/cplus/SkewHeap.h
[link_skewheap_cplus_02]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/cplus/SkewHeapTest.cpp
[link_leftist_cplus]: /2013/03/02/leftist-cplus/

