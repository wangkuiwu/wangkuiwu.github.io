---
layout: post
title: "伸展树(三)之 Java语言详解"
description: "tree"
category: datastructure
tags: [tree,java]
date: 2013-02-03 18:18
---
 
> 前面分别通过C和C++实现了伸展树，本章给出伸展树的Java版本。基本算法和原理都与前两章一样。

> 目录  
[第1部分 伸展树的介绍](#anchor1)  
[第2部分 伸展树的Java实现](#anchor2)  
[第3部分 伸展树的Java实现(完整源码)](#anchor3)  



<a name="anchor1"></a>
# 第1部分 伸展树的介绍

伸展树(Splay Tree)是特殊的二叉查找树。

它的特殊是指，它除了本身是棵二叉查找树之外，它还具备一个特点: 当某个节点被访问时，伸展树会通过旋转使该节点成为树根。这样做的好处是，下次要访问该节点时，能够迅速的访问到该节点。

 
<a name="anchor2"></a>
# 第2部分 伸展树的Java实现

## 1. 基本定义

    public class SplayTree<T extends Comparable<T>> {

        private SplayTreeNode<T> mRoot;    // 根结点

        public class SplayTreeNode<T extends Comparable<T>> {
            T key;                // 关键字(键值)
            SplayTreeNode<T> left;    // 左孩子
            SplayTreeNode<T> right;    // 右孩子

            public SplayTreeNode() {
                this.left = null;
                this.right = null;
            }

            public SplayTreeNode(T key, SplayTreeNode<T> left, SplayTreeNode<T> right) {
                this.key = key;
                this.left = left;
                this.right = right;
            }
        }

            ...
    }

SplayTree是伸展树，而SplayTreeNode是伸展树节点。在此，我将SplayTreeNode定义为SplayTree的内部类。在伸展树SplayTree中包含了伸展树的根节点mRoot。SplayTreeNode包括的几个组成元素:  
(01) key -- 是关键字，是用来对伸展树的节点进行排序的。  
(02) left -- 是左孩子。  
(03) right -- 是右孩子。

 

## 2. 旋转

旋转是伸展树中需要重点关注的，它的代码如下：

    /* 
     * 旋转key对应的节点为根节点，并返回根节点。
     *
     * 注意：
     *   (a)：伸展树中存在"键值为key的节点"。
     *          将"键值为key的节点"旋转为根节点。
     *   (b)：伸展树中不存在"键值为key的节点"，并且key < tree.key。
     *      b-1 "键值为key的节点"的前驱节点存在的话，将"键值为key的节点"的前驱节点旋转为根节点。
     *      b-2 "键值为key的节点"的前驱节点存在的话，则意味着，key比树中任何键值都小，那么此时，将最小节点旋转为根节点。
     *   (c)：伸展树中不存在"键值为key的节点"，并且key > tree.key。
     *      c-1 "键值为key的节点"的后继节点存在的话，将"键值为key的节点"的后继节点旋转为根节点。
     *      c-2 "键值为key的节点"的后继节点不存在的话，则意味着，key比树中任何键值都大，那么此时，将最大节点旋转为根节点。
     */
    private SplayTreeNode<T> splay(SplayTreeNode<T> tree, T key) {
        if (tree == null) 
            return tree;

        SplayTreeNode<T> N = new SplayTreeNode<T>();
        SplayTreeNode<T> l = N;
        SplayTreeNode<T> r = N;
        SplayTreeNode<T> c;

        for (;;) {

            int cmp = key.compareTo(tree.key);
            if (cmp < 0) {

                if (tree.left == null)
                    break;

                if (key.compareTo(tree.left.key) < 0) {
                    c = tree.left;                           /* rotate right */
                    tree.left = c.right;
                    c.right = tree;
                    tree = c;
                    if (tree.left == null) 
                        break;
                }
                r.left = tree;                               /* link right */
                r = tree;
                tree = tree.left;
            } else if (cmp > 0) {

                if (tree.right == null) 
                    break;

                if (key.compareTo(tree.right.key) > 0) {
                    c = tree.right;                          /* rotate left */
                    tree.right = c.left;
                    c.left = tree;
                    tree = c;
                    if (tree.right == null) 
                        break;
                }

                l.right = tree;                              /* link left */
                l = tree;
                tree = tree.right;
            } else {
                break;
            }
        }

        l.right = tree.left;                                /* assemble */
        r.left = tree.right;
        tree.left = N.right;
        tree.right = N.left;

        return tree;
    }

    public void splay(T key) {
        mRoot = splay(mRoot, key);
    }

上面的代码的作用：将"键值为key的节点"旋转为根节点，并返回根节点。它的处理情况共包括：  
(a)：伸展树中存在"键值为key的节点"。  
&nbsp;&nbsp;&nbsp;&nbsp; 将"键值为key的节点"旋转为根节点。  
(b)：伸展树中不存在"键值为key的节点"，并且key < tree->key。  
&nbsp;&nbsp;&nbsp;&nbsp; b-1) "键值为key的节点"的前驱节点存在的话，将"键值为key的节点"的前驱节点旋转为根节点。  
&nbsp;&nbsp;&nbsp;&nbsp; b-2) "键值为key的节点"的前驱节点存在的话，则意味着，key比树中任何键值都小，那么此时，将最小节点旋转为根节点。  
(c)：伸展树中不存在"键值为key的节点"，并且key > tree->key。  
&nbsp;&nbsp;&nbsp;&nbsp; c-1) "键值为key的节点"的后继节点存在的话，将"键值为key的节点"的后继节点旋转为根节点。  
&nbsp;&nbsp;&nbsp;&nbsp; c-2) "键值为key的节点"的后继节点不存在的话，则意味着，key比树中任何键值都大，那么此时，将最大节点旋转为根节点。


下面列举个例子分别对a进行说明。

在下面的伸展树中查找10，共包括"右旋"  --> "右链接"  --> "组合"这3步。

![img](/media/pic/datastruct_algrithm/tree/splaytree/splaytree_00.jpg)
 

**第一步： 右旋**  
对应代码中的"rotate right"部分

![img](/media/pic/datastruct_algrithm/tree/splaytree/splaytree_01.jpg)
 

**第二步： 右链接**  
对应代码中的"link right"部分

![img](/media/pic/datastruct_algrithm/tree/splaytree/splaytree_02.jpg)
 

**第三步： 组合**
对应代码中的"assemble"部分

![img](/media/pic/datastruct_algrithm/tree/splaytree/splaytree_03.jpg)

提示：如果在上面的伸展树中查找"70"，则正好与"示例1"对称，而对应的操作则分别是"rotate left", "link left"和"assemble"。  
其它的情况，例如"查找15是b-1的情况，查找5是b-2的情况"等等，这些都比较简单，大家可以自己分析。




## 3. 插入

插入代码

    /* 
    * 将结点插入到伸展树中，并返回根节点
     *
     * 参数说明：
     *     tree 伸展树的
     *     z 插入的结点
     */
    private SplayTreeNode<T> insert(SplayTreeNode<T> tree, SplayTreeNode<T> z) {
        int cmp;
        SplayTreeNode<T> y = null;
        SplayTreeNode<T> x = tree;

        // 查找z的插入位置
        while (x != null) {
            y = x;
            cmp = z.key.compareTo(x.key);
            if (cmp < 0)
                x = x.left;
            else if (cmp > 0)
                x = x.right;
            else {
                System.out.printf("不允许插入相同节点(%d)!\n", z.key);
                z=null;
                return tree;
            }
        }

        if (y==null)
            tree = z;
        else {
            cmp = z.key.compareTo(y.key);
            if (cmp < 0)
                y.left = z;
            else
                y.right = z;
        }

        return tree;
    }

    public void insert(T key) {
        SplayTreeNode<T> z=new SplayTreeNode<T>(key,null,null);

        // 如果新建结点失败，则返回。
        if ((z=new SplayTreeNode<T>(key,null,null)) == null)
            return ;

        // 插入节点
        mRoot = insert(mRoot, z);
        // 将节点(key)旋转为根节点
        mRoot = splay(mRoot, key);
    }

insert(key)是提供给外部的接口，它的作用是新建节点(节点的键值为key)，并将节点插入到伸展树中；然后，将该节点旋转为根节点。  
insert(tree, z)是内部接口，它的作用是将节点z插入到tree中。insert(tree, z)在将z插入到tree中时，仅仅只将tree当作是一棵二叉查找树，而且不允许插入相同节点。

 

## 4. 删除

删除代码

    /* 
     * 删除结点(z)，并返回被删除的结点
     *
     * 参数说明：
     *     bst 伸展树
     *     z 删除的结点
     */
    private SplayTreeNode<T> remove(SplayTreeNode<T> tree, T key) {
        SplayTreeNode<T> x;

        if (tree == null) 
            return null;

        // 查找键值为key的节点，找不到的话直接返回。
        if (search(tree, key) == null)
            return tree;

        // 将key对应的节点旋转为根节点。
        tree = splay(tree, key);

        if (tree.left != null) {
            // 将"tree的前驱节点"旋转为根节点
            x = splay(tree.left, key);
            // 移除tree节点
            x.right = tree.right;
        }
        else
            x = tree.right;

        tree = null;

        return x;
    }

    public void remove(T key) {
        mRoot = remove(mRoot, key);
    }

remove(key)是外部接口，remove(tree, key)是内部接口。  
remove(tree, key)的作用是：删除伸展树中键值为key的节点。  
它会先在伸展树中查找键值为key的节点。若没有找到的话，则直接返回。若找到的话，则将该节点旋转为根节点，然后再删除该节点。


关于"前序遍历"、"中序遍历"、"后序遍历"、"最大值"、"最小值"、"查找"、"打印伸展树"、"销毁伸展树"等接口就不再单独介绍了，Please RTFSC(Read The Fucking Source Code)！这些接口，与前面介绍的"二叉查找树"、"AVL树"的相关接口都是类似的。

 
<a name="anchor3"></a>
# 第3部分 伸展树的Java实现(完整源码)


点击查看：[源代码][link_source_code]

在二叉查找树的Java实现中，使用了泛型，也就意味着它支持任意类型；但是该类型必须要实现Comparable接口。

 
伸展树的测试程序运行结果如下：

    == 依次添加: 10 50 40 30 20 60 
    == 前序遍历: 60 30 20 10 50 40 
    == 中序遍历: 10 20 30 40 50 60 
    == 后序遍历: 10 20 40 50 30 60 
    == 最小值: 10
    == 最大值: 60
    == 树的详细信息: 
    60 is root
    30 is 60's   left child
    20 is 30's   left child
    10 is 20's   left child
    50 is 30's  right child
    40 is 50's   left child

    == 旋转节点(30)为根节点
    == 树的详细信息: 
    30 is root
    20 is 30's   left child
    10 is 20's   left child
    60 is 30's  right child
    50 is 60's   left child
    40 is 50's   left child

测试程序的主要流程是：新建伸展树，然后向伸展树中依次插入10,50,40,30,20,60。插入完毕这些数据之后，伸展树的节点是60；此时，再旋转节点，使得30成为根节点。
依次插入10,50,40,30,20,60示意图如下：

 
![img](/media/pic/datastruct_algrithm/tree/splaytree/splaytree_04.jpg)

将30旋转为根节点的示意图如下：

![img](/media/pic/datastruct_algrithm/tree/splaytree/splaytree_05.jpg)

 
[link_source_code]: https://github.com/wangkuiwu/datastructs_and_algorithm/tree/master/source/tree/splaytree/java
