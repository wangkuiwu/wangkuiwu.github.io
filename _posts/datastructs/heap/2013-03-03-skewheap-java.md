---
layout: post
title: "斜堆(三)之 Java详解"
description: "skew heap"
category: datastructure
tags: [heap,java]
date: 2013-03-03 18:11
---


> 前面分别通过[C][link_skewheap_c]和[C++][link_skewheap_cplus]实现了斜堆，本章给出斜堆的java版本。



# 斜堆的介绍

斜堆(Skew heap)也叫自适应堆(self-adjusting heap)，它是左倾堆的一个变种。它的合并操作的时间复杂度也是O(log n)。

相比于[左倾堆][link_leftist_java]，斜堆的节点没有"零距离"这个属性。除此之外，它们斜堆的合并操作也不同。斜堆的合并操作算法如下：

> 1. 如果一个空斜堆与一个非空斜堆合并，返回非空斜堆。  
> 2. 如果两个斜堆都非空，那么比较两个根节点，取较小堆的根节点为新的根节点。将"较小堆的根节点的右孩子"和"较大堆"进行合并。  
> 3. 合并后，交换新堆根节点的左孩子和右孩子。  

第3步是斜堆和左倾堆的合并操作差别的关键所在，如果是左倾堆，则合并后要比较左右孩子的零距离大小，若右孩子的零距离 > 左孩子的零距离，则交换左右孩子；最后，在设置根的零距离。


# 斜堆的基本操作

## 1. 基本定义

    public class SkewHeap<T extends Comparable<T>> {

        private SkewNode<T> mRoot;    // 根结点

        private class SkewNode<T extends Comparable<T>> {
            T key;                // 关键字(键值)
            SkewNode<T> left;    // 左孩子
            SkewNode<T> right;    // 右孩子

            public SkewNode(T key, SkewNode<T> left, SkewNode<T> right) {
                this.key = key;
                this.left = left;
                this.right = right;
            }

            public String toString() {
                return "key:"+key;
            }
        }

        ...
    }

SkewNode是斜堆对应的节点类。
SkewHeap是斜堆类，它包含了斜堆的根节点，以及斜堆的操作。


## 2. 合并

    /*
     * 合并"斜堆x"和"斜堆y"
     */
    private SkewNode<T> merge(SkewNode<T> x, SkewNode<T> y) {
        if(x == null) return y;
        if(y == null) return x;

        // 合并x和y时，将x作为合并后的树的根；
        // 这里的操作是保证: x的key < y的key
        if(x.key.compareTo(y.key) > 0) {
            SkewNode<T> tmp = x;
            x = y;
            y = tmp;
        }

        // 将x的右孩子和y合并，
        // 合并后直接交换x的左右孩子，而不需要像左倾堆一样考虑它们的npl。
        SkewNode<T> tmp = merge(x.right, y);
        x.right = x.left;
        x.left = tmp;

        return x;
    }

    public void merge(SkewHeap<T> other) {
        this.mRoot = merge(this.mRoot, other.mRoot);
    }

merge(x, y)是内部接口，作用是合并x和y这两个斜堆，并返回得到的新堆的根节点。

merge(other)是外部接口，作用是将other合并到当前堆中。



## 3. 添加

    /* 
     * 新建结点(key)，并将其插入到斜堆中
     *
     * 参数说明：
     *     key 插入结点的键值
     */
    public void insert(T key) {
        SkewNode<T> node = new SkewNode<T>(key,null,null);

        // 如果新建结点失败，则返回。
        if (node != null)
            this.mRoot = merge(this.mRoot, node);
    }

insert(key)的作用是新建键值为key的节点，并将其加入到当前斜堆中。

## 4. 删除

    /* 
     * 删除根结点
     * 
     * 返回值：
     *     返回被删除的节点的键值
     */
    public T remove() {
        if (this.mRoot == null)
            return null;

        T key = this.mRoot.key;
        SkewNode<T> l = this.mRoot.left;
        SkewNode<T> r = this.mRoot.right;

        this.mRoot = null;          // 删除根节点
        this.mRoot = merge(l, r);   // 合并左右子树

        return key;
    }

remove()的作用是删除斜堆的最小节点。

<br/>
*PS. 注意：关于斜堆的"前序遍历"、"中序遍历"、"后序遍历"、"打印"、"销毁"等接口就不再单独介绍了。后文的源码中有给出它们的实现代码，Please RTFSC(Read The Fucking Source Code)！*




# 斜堆的完整源码

包含测试程序在内，一共2个文件。

1. [斜堆的实现文件(SkewHeap.java)][link_skewheap_java_01] 

2. [斜堆的测试程序(SkewHeapTest.java)][link_skewheap_java_02] 



[link_skewheap_java_01]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/java/SkewHeap.java
[link_skewheap_java_02]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/heap/skewheap/java/SkewHeapTest.java
[link_leftist_java]: /2013/03/02/leftist-java/
[link_skewheap_c]: /2013/03/03/skewheap-c/
[link_skewheap_cplus]: /2013/03/03/skewheap-cplus/

