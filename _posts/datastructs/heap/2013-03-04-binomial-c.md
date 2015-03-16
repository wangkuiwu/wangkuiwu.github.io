---
layout: post
title: "二项堆(一)之 C语言详解"
description: "heap"
category: datastructure
tags: [heap]
date: 2013-03-04 09:02
---
 
> 本章介绍二项堆，它和之前所讲的堆(二叉堆、左倾堆、斜堆)一样，也是用于实现优先队列的。和以往一样，本文会先对二项堆的理论知识进行简单介绍，然后给出C语言的实现。后续再分别给出C++和Java版本的实现；实现的语言虽不同，但是原理一样，选择其中之一进行了解即可。若文章有错误或不足的地方，请不吝指出！

> **目录**  
[第1部分 二项树的介绍](#anchor1)  
[第2部分 二项堆的介绍](#anchor2)  
[第3部分 二项堆的基本操作](#anchor3)  
[第4部分 二项堆的C实现(完整源码)](#anchor4)  


<a name="anchor1"></a>
# 第1部分 二项树的介绍

**二项树的定义**

二项堆是二项树的集合。在了解二项堆之前，先对二项树进行介绍。

二项树是一种递归定义的有序树。它的递归定义如下：  
(01) 二项树B<sub>0</sub>只有一个结点；  
(02) 二项树B<sub>k</sub>由两棵二项树B<sub>(k-1)</sub>组成的，其中一棵树是另一棵树根的最左孩子。

如下图所示：

![img](/media/pic/datastruct_algrithm/heap/binomial/01.jpg)

上图的B<sub>0</sub>、B<sub>1</sub>、B<sub>2</sub>、B<sub>3</sub>、B<sub>4</sub>都是二项树。对比前面提到的二项树的定义：B<sub>0</sub>只有一个节点，B<sub>1</sub>由两个B<sub>0</sub>所组成，B<sub>2</sub>由两个B<sub>1</sub>所组成，B<sub>3</sub>由两个B<sub>2</sub>所组成，B<sub>4</sub>由两个B<sub>3</sub>所组成；而且，当两颗相同的二项树组成另一棵树时，其中一棵树是另一棵树的最左孩子。

 

**二项树的性质**

二项树有以下性质：  
**[性质一]** B<sub>k</sub>共有2<sup>k</sup>个节点。  
**[性质二]** B<sub>k</sub>的高度为k。  
**[性质三]** B<sub>k</sub>在深度i处恰好有C(k,i)个节点，其中i=0,1,2,...,k。  
**[性质四]** 根的度数为k，它大于任何其它节点的度数。  
注意：树的高度和深度是相同的。关于树的高度的概念，《算法导论》中只有一个节点的树的高度是0，而"维基百科"中只有一个节点的树的高度是1。本文使用了《算法导论中》"树的高度和深度"的概念。

下面对这几个性质进行简单说明：  
[性质一] B<sub>k</sub>共有2<sup>k</sup>个节点。  
   如上图所示，B<sub>0</sub>有2<sup>0</sup>=1节点，B<sub>1</sub>有2<sup>1</sup>=2个节点，B<sub>2</sub>有2<sup>2</sup>=4个节点，...
[性质二] B<sub>k</sub>的高度为k。  
   &nbsp;&nbsp;&nbsp;&nbsp; 如上图所示，B<sub>0</sub>的高度为0，B<sub>1</sub>的高度为1，B<sub>2</sub>的高度为2，...  
[性质三] B<sub>k</sub>在深度i处恰好有C(k,i)个节点，其中i=0,1,2,...,k。  
  &nbsp;&nbsp;&nbsp;&nbsp; C(k,i)是高中数学中阶乘元素，例如，C(10,3)=(10*9*8) / (3*2*1)=240  
  &nbsp;&nbsp;&nbsp;&nbsp; B<sub>4</sub>中深度为0的节点C(4,0)=1  
  &nbsp;&nbsp;&nbsp;&nbsp; B<sub>4</sub>中深度为1的节点C(4,1)= 4 / 1 = 4  
  &nbsp;&nbsp;&nbsp;&nbsp; B<sub>4</sub>中深度为2的节点C(4,2)= (4*3) / (2*1) = 6  
  &nbsp;&nbsp;&nbsp;&nbsp; B<sub>4</sub>中深度为3的节点C(4,3)= (4*3*2) / (3*2*1) = 4  
  &nbsp;&nbsp;&nbsp;&nbsp; B<sub>4</sub>中深度为4的节点C(4,4)= (4*3*2*1) / (4*3*2*1) = 1  
&nbsp;&nbsp;&nbsp;&nbsp;  合计得到B<sub>4</sub>的节点分布是(1,4,6,4,1)。  
[性质四] 根的度数为k，它大于任何其它节点的度数。  
 &nbsp;&nbsp;&nbsp;&nbsp;  节点的度数是该结点拥有的子树的数目。

 
<a name="anchor2"></a>
# 第2部分 二项堆的介绍

二项堆通常被用来实现优先队列，它堆是指满足以下性质的二项树的集合：  
(01) 每棵二项树都满足最小堆性质。即，父节点的关键字 <= 它的孩子的关键字。  
(02) 不能有两棵或以上的二项树具有相同的度数(包括度数为0)。换句话说，具有度数k的二项树有0个或1个。

![img](/media/pic/datastruct_algrithm/heap/binomial/02.jpg)
 
上图就是一棵二项堆，它由二项树B<sub>0</sub>、B<sub>2</sub>和B<sub>3</sub>组成。对比二项堆的定义：(01)二项树B<sub>0</sub>、B<sub>2</sub>、B<sub>3</sub>都是最小堆；(02)二项堆不包含相同度数的二项树。

二项堆的第(01)个性质保证了二项堆的最小节点是某一棵二项树的根节点，第(02)个性质则说明结点数为n的二项堆最多只有log{n} + 1棵二项树。实际上，将包含n个节点的二项堆，表示成若干个2的指数和(或者转换成二进制)，则每一个2个指数都对应一棵二项树。例如，13(二进制是1101)的2个指数和为13=2<sup>3</sup> + 2<sup>2</sup> + 2<sup>0</sup>, 因此具有13个节点的二项堆由度数为3, 2, 0的三棵二项树组成。

 
<a name="anchor3"></a>
# 第3部分 二项堆的基本操作

二项堆是可合并堆，它的合并操作的复杂度是O(log n)。

## 1. 基本定义

    #ifndef _BINOMIAL_HEAP_H_
    #define _BINOMIAL_HEAP_H_

    typedef int Type;

    typedef struct _BinomialNode{
        Type   key;                     // 关键字(键值)
        int degree;                     // 度数
        struct _BinomialNode *child;    // 左孩子
        struct _BinomialNode *parent;   // 父节点
        struct _BinomialNode *next;     // 兄弟
    }BinomialNode, *BinomialHeap;

    // 新建key对应的节点，并将其插入到二项堆中。
    BinomialNode* binomial_insert(BinomialHeap heap, Type key);
    // 删除节点：删除键值为key的节点，并返回删除节点后的二项树
    BinomialNode* binomial_delete(BinomialHeap heap, Type key);
    // 将二项堆heap的键值oldkey更新为newkey
    void binomial_update(BinomialHeap heap, Type oldkey, Type newkey);

    // 合并二项堆：将h1, h2合并成一个堆，并返回合并后的堆
    BinomialNode* binomial_union(BinomialHeap h1, BinomialHeap h2) ;

    // 查找：在二项堆中查找键值为key的节点
    BinomialNode* binomial_search(BinomialHeap heap, Type key);
    // 获取二项堆中的最小节点
    BinomialNode* binomial_minimum(BinomialHeap heap) ;
    // 移除最小节点，并返回移除节点后的二项堆
    BinomialNode* binomial_extract_minimum(BinomialHeap heap);

    // 打印"二项堆"
    void binomial_print(BinomialHeap heap);

    #endif

BinomialNode是二项堆的节点。它包括了关键字(key)，用于比较节点大小；度数(degree)，用来表示当前节点的度数；左孩子(child)、父节点(parent)以及兄弟节点(next)。  
下面是一棵二项堆的树形图和它对应的内存结构关系图。

![img](/media/pic/datastruct_algrithm/heap/binomial/03.jpg)

 

 

## 2. 合并操作

合并操作是二项堆的重点，二项堆的添加操作也是基于合并操作来实现的。

合并两个二项堆，需要的步骤概括起来如下：  
(01) 将两个二项堆的根链表合并成一个链表。合并后的新链表按照"节点的度数"单调递增排列。  
(02) 将新链表中"根节点度数相同的二项树"连接起来，直到所有根节点度数都不相同。

下面，先看看合并操作的代码；然后再通过示意图对合并操作进行说明。

binomial_merge()代码(C语言)

    /*
     * 将h1, h2中的根表合并成一个按度数递增的链表，返回合并后的根节点
     */
    static BinomialNode* binomial_merge(BinomialHeap h1, BinomialHeap h2) 
    {
        BinomialNode* head = NULL; //heap为指向新堆根结点
        BinomialNode** pos = &head;

        while (h1 && h2)
        {
            if (h1->degree < h2->degree)
            {
                *pos = h1;
                h1 = h1->next;
            } 
            else 
            {
                *pos = h2;
                h2 = h2->next;
            }
            pos = &(*pos)->next;
        }
        if (h1)
            *pos = h1;
        else
            *pos = h2;

        return head;
    }

binomial_link()代码(C语言)

    /*
     * 合并两个二项堆：将child合并到heap中
     */
    static void binomial_link(BinomialHeap child, BinomialHeap heap) 
    {
        child->parent = heap;
        child->next   = heap->child;
        heap->child = child;
        heap->degree++;
    }

合并操作代码(C语言)

    /*
     * 合并二项堆：将h1, h2合并成一个堆，并返回合并后的堆
     */
    BinomialNode* binomial_union(BinomialHeap h1, BinomialHeap h2) 
    {
        BinomialNode *heap;
        BinomialNode *prev_x, *x, *next_x;

        // 将h1, h2中的根表合并成一个按度数递增的链表heap
        heap = binomial_merge(h1, h2);
        if (heap == NULL)
            return NULL;
     
        prev_x = NULL;
        x      = heap;
        next_x = x->next;
     
        while (next_x != NULL)
        {
            if (   (x->degree != next_x->degree) 
                || ((next_x->next != NULL) && (next_x->degree == next_x->next->degree))) 
            {
                // Case 1: x->degree != next_x->degree
                // Case 2: x->degree == next_x->degree == next_x->next->degree
                prev_x = x;
                x = next_x;
            } 
            else if (x->key <= next_x->key) 
            {
                // Case 3: x->degree == next_x->degree != next_x->next->degree
                //      && x->key    <= next_x->key
                x->next = next_x->next;
                binomial_link(next_x, x);
            } 
            else 
            {
                // Case 4: x->degree == next_x->degree != next_x->next->degree
                //      && x->key    >  next_x->key
                if (prev_x == NULL) 
                {
                    heap = next_x;
                } 
                else 
                {
                    prev_x->next = next_x;
                }
                binomial_link(x, next_x);
                x = next_x;
            }
            next_x = x->next;
        }

        return heap;
    }

合并函数binomial_union(h1, h2)的作用是将h1和h2合并，并返回合并后的二项堆。在binomial_union(h1, h2)中，涉及到了两个函数binomial_merge(h1, h2)和binomial_link(child, heap)。  
binomial_merge(h1, h2)就是我们前面所说的"两个二项堆的根链表合并成一个链表，合并后的新链表按照'节点的度数'单调递增排序"。  
binomial_link(child, heap)则是为了合并操作的辅助函数，它的作用是将"二项堆child的根节点"设为"二项堆heap的左孩子"，从而将child整合到heap中去。


在binomial_union(h1, h2)中对h1和h2进行合并时；首先通过 binomial_merge(h1, h2) 将h1和h2的根链表合并成一个"按节点的度数单调递增"的链表；然后进入while循环，对合并得到的新链表进行遍历，将新链表中"根节点度数相同的二项树"连接起来，直到所有根节点度数都不相同为止。在将新联表中"根节点度数相同的二项树"连接起来时，可以将被连接的情况概括为4种。

    x是根链表的当前节点，next_x是x的下一个(兄弟)节点。
    Case 1: x->degree != next_x->degree
                 即，"当前节点的度数"与"下一个节点的度数"相等时。此时，不需要执行任何操作，继续查看后面的节点。
    Case 2: x->degree == next_x->degree == next_x->next->degree
                 即，"当前节点的度数"、"下一个节点的度数"和"下下一个节点的度数"都相等时。此时，暂时不执行任何操作，还是继续查看后面的节点。实际上，这里是将"下一个节点"和"下下一个节点"等到后面再进行整合连接。
    Case 3: x->degree == next_x->degree != next_x->next->degree
            && x->key <= next_x->key
                 即，"当前节点的度数"与"下一个节点的度数"相等，并且"当前节点的键值"<="下一个节点的度数"。此时，将"下一个节点(对应的二项树)"作为"当前节点(对应的二项树)的左孩子"。
    Case 4: x->degree == next_x->degree != next_x->next->degree
            && x->key > next_x->key
                 即，"当前节点的度数"与"下一个节点的度数"相等，并且"当前节点的键值">"下一个节点的度数"。此时，将"当前节点(对应的二项树)"作为"下一个节点(对应的二项树)的左孩子"。


下面通过示意图来对合并操作进行说明。

![img](/media/pic/datastruct_algrithm/heap/binomial/04.jpg)

第1步：将两个二项堆的根链表合并成一个链表  
 &nbsp;&nbsp;&nbsp;&nbsp;  执行完第1步之后，得到的新链表中有许多度数相同的二项树。实际上，此时得到的是对应"Case 4"的情况，"树41"(根节点为41的二项树)和"树13"的度数相同，且"树41"的键值 > "树13"的键值。此时，将"树41"作为"树13"的左孩子。  
第2步：合并"树41"和"树13"  
 &nbsp;&nbsp;&nbsp;&nbsp; 执行完第2步之后，得到的是对应"Case 3"的情况，"树13"和"树28"的度数相同，且"树13"的键值 < "树28"的键值。此时，将"树28"作为"树13"的左孩子。  
第3步：合并"树13"和"树28"  
 &nbsp;&nbsp;&nbsp;&nbsp; 执行完第3步之后，得到的是对应"Case 2"的情况，"树13"、"树28"和"树7"这3棵树的度数都相同。此时，将x设为下一个节点。  
第4步：将x和next_x往后移  
 &nbsp;&nbsp;&nbsp;&nbsp; 执行完第4步之后，得到的是对应"Case 3"的情况，"树7"和"树11"的度数相同，且"树7"的键值 < "树11"的键值。此时，将"树11"作为"树7"的左孩子。  
第5步：合并"树7"和"树11"  
 &nbsp;&nbsp;&nbsp;&nbsp; 执行完第5步之后，得到的是对应"Case 4"的情况，"树7"和"树6"的度数相同，且"树7"的键值 > "树6"的键值。此时，将"树7"作为"树6"的左孩子。  
第6步：合并"树7"和"树6"  
 此时，合并操作完成！

PS. 合并操作的图文解析过程与"二项堆的测试程序(main.c)中的test_union()函数"是对应的！

 

## 3. 插入操作

理解了"合并"操作之后，插入操作就相当简单了。插入操作可以看作是将"要插入的节点"和当前已有的堆进行合并。

插入操作代码(C语言)

    /*
     * 新建key对应的节点，并将其插入到二项堆中。
     *
     * 参数说明：
     *     heap -- 原始的二项树。
     *     key -- 键值
     * 返回值：
     *     插入key之后的二项树
     */
    BinomialNode* binomial_insert(BinomialHeap heap, Type key)
    {
        BinomialNode* node;

        if (binomial_search(heap, key) != NULL)
        {
            printf("insert failed: the key(%d) is existed already!\n", key);
            return heap;
        }

        node = make_binomial_node(key);
        if (node==NULL)
            return heap;

        return binomial_union(heap, node);
    }

在插入时，首先通过binomial_search(heap, key)查找键值为key的节点。存在的话，则直接返回；不存在的话，则通过make_binomial_node(key)新建键值为key的节点node，然后将node和heap进行合并。

注意：我这里实现的二项堆是"进制插入相同节点的"！若你想允许插入相同键值的节点，则屏蔽掉插入操作中的binomial_search(heap, key)部分代码即可。

 

## 4. 删除操作

删除二项堆中的某个节点，需要的步骤概括起来如下：  
(01) 将"该节点"交换到"它所在二项树"的根节点位置。方法是，从"该节点"不断向上(即向树根方向)"遍历，不断交换父节点和子节点的数据，直到被删除的键值到达树根位置。  
(02) 将"该节点所在的二项树"从二项堆中移除；将该二项堆记为heap。  
(03) 将"该节点所在的二项树"进行反转。反转的意思，就是将根的所有孩子独立出来，并将这些孩子整合成二项堆，将该二项堆记为child。  
(04) 将child和heap进行合并操作。

下面，先看看删除操作的代码；再进行图文说明。
binomial_reverse()代码(C语言)

    /*
     * 反转二项堆heap
     */
    static BinomialNode* binomial_reverse(BinomialNode* heap)
    {
        BinomialNode* next;
        BinomialNode* tail = NULL;

        if (!heap)
            return heap;

        heap->parent = NULL;
        while (heap->next) 
        {
            next          = heap->next;
            heap->next = tail;
            tail          = heap;
            heap          = next;
            heap->parent  = NULL;
        }
        heap->next = tail;

        return heap;
    }

删除操作代码(C语言)

    /* 
     * 删除节点：删除键值为key的节点，并返回删除节点后的二项树
     */
    BinomialNode* binomial_delete(BinomialHeap heap, Type key)
    {
        BinomialNode *node;
        BinomialNode *parent, *prev, *pos;

        if (heap==NULL)
            return heap;

        // 查找键值为key的节点
        if ((node = binomial_search(heap, key)) == NULL)
            return heap;

        // 将被删除的节点的数据数据上移到它所在的二项树的根节点
        parent = node->parent;
        while (parent != NULL)
        {
            // 交换数据
            swap(node->key, parent->key);
            // 下一个父节点
            node   = parent;
            parent = node->parent;
        }

        // 找到node的前一个根节点(prev)
        prev = NULL;
        pos  = heap;
        while (pos != node) 
        {
            prev = pos;
            pos  = pos->next;
        }
        // 移除node节点
        if (prev)
            prev->next = node->next;
        else
            heap = node->next;

        heap = binomial_union(heap, binomial_reverse(node->child)); 

        free(node);

        return heap;
    }

binomial_delete(heap, key)的作用是删除二项堆heap中键值为key的节点，并返回删除节点后的二项堆。  
binomial_reverse(heap)的作用是反转二项堆heap，并返回反转之后的根节点。


下面通过示意图来对删除操作进行说明(删除二项堆中的节点20)。

![img](/media/pic/datastruct_algrithm/heap/binomial/05.jpg)

总的思想，就是将被"删除节点"从它所在的二项树中孤立出来，然后再对二项树进行相应的处理。

PS. 删除操作的图文解析过程与"二项堆的测试程序(main.c)中的test_delete()函数"是对应的！

 

## 5. 更新操作

更新二项堆中的某个节点，就是修改节点的值，它包括两部分分："减少节点的值" 和 "增加节点的值" 。

更新操作代码(C语言)

    /* 
     * 更新二项堆heap的节点node的键值为key
     */
    static void binomial_update_key(BinomialHeap heap, BinomialNode* node, Type key)
    {
        if (node == NULL)
            return ;

        if(key < node->key)
            binomial_decrease_key(heap, node, key);
        else if(key > node->key)
            binomial_increase_key(heap, node, key);
        else
            printf("No need to update!!!\n");
    }

 

### 5.1 减少节点的值

减少节点值的操作很简单：该节点一定位于一棵二项树中，减小"二项树"中某个节点的值后要保证"该二项树仍然是一个最小堆"；因此，就需要我们不断的将该节点上调。

减少操作代码(C语言)

    /* 
     * 减少关键字的值：将二项堆heap中的节点node的键值减小为key。
     */
    static void binomial_decrease_key(BinomialHeap heap, BinomialNode *node, Type key)
    {
        if ((key >= node->key) || (binomial_search(heap, key) != NULL))
        {
            printf("decrease failed: the new key(%d) is existed already, \
                    or is no smaller than current key(%d)\n", key, node->key);
            return ;
        }
        node->key = key;
     
        BinomialNode *child, *parent;
        child = node;
        parent = node->parent;
        while(parent != NULL && child->key < parent->key)
        {
            swap(parent->key, child->key);
            child = parent;
            parent = child->parent;
        }
    }

 

下面是减少操作的示意图(20->2)

![img](/media/pic/datastruct_algrithm/heap/binomial/06.jpg)

减少操作的思想很简单，就是"保持被减节点所在二项树的最小堆性质"。

PS. 减少操作的图文解析过程与"二项堆的测试程序(main.c)中的test_decrease()函数"是对应的！

 

### 5.2 增加节点的值

增加节点值的操作也很简单。上面说过减少要将被减少的节点不断上调，从而保证"被减少节点所在的二项树"的最小堆性质；而增加操作则是将被增加节点不断的下调，从而保证"被增加节点所在的二项树"的最小堆性质。

增加操作代码(C语言)

    /* 
     * 增加关键字的值：将二项堆heap中的节点node的键值增加为key。
     */
    static void binomial_increase_key(BinomialHeap heap, BinomialNode *node, Type key)
    {
        if ((key <= node->key) || (binomial_search(heap, key) != NULL))
        {
            printf("increase failed: the new key(%d) is existed already, \
                    or is no greater than current key(%d)\n", key, node->key);
            return ;
        }
        node->key = key;

        BinomialNode *cur, *child, *least;
        cur = node;
        child = cur->child;
        while (child != NULL) 
        {
            if(cur->key > child->key)
            {
                // 如果"当前节点" < "它的左孩子"，
                // 则在"它的孩子中(左孩子 和 左孩子的兄弟)"中，找出最小的节点；
                // 然后将"最小节点的值" 和 "当前节点的值"进行互换
                least = child;
                while(child->next != NULL)
                {
                    if (least->key > child->next->key)
                    {
                        least = child->next;
                    }
                    child = child->next;
                }
                // 交换最小节点和当前节点的值
                swap(least->key, cur->key);

                // 交换数据之后，再对"原最小节点"进行调整，使它满足最小堆的性质：父节点 <= 子节点
                cur = least;
                child = cur->child;
            }
            else
            {
                child = child->next;
            }
        }
    }

 

下面是增加操作的示意图(6->60)

![img](/media/pic/datastruct_algrithm/heap/binomial/07.jpg)

增加操作的思想很简单，"保持被增加点所在二项树的最小堆性质"。

PS. 增加操作的图文解析过程与"二项堆的测试程序(main.c)中的test_increase()函数"是对应的！

 

注意：**关于二项堆的"查找"、"打印"等其它接口就不再单独介绍了，后文的源码中有给出它们的实现代码。有兴趣的话，Please RTFSC(Read The Fucking Source Code)！**

 
<a name="anchor4"></a>
# 第4部分 二项堆的C实现(完整源码)

点击查看：[源代码][link_source_code]

 
二项堆的测试程序包括了五部分，分别是"插入"、"删除"、"增加"、"减少"、"合并"这5种功能的测试代码。默认是运行的"插入"功能代码，你可以根据自己的需要来对相应的功能进行验证！

下面是插入功能运行结果：

    == 二项堆(ha)中依次添加: 12 7 25 15 28 33 41 
    == 二项堆(ha)的详细信息: 
    == 二项堆( B0 B1 B2 )的详细信息：
    1. 二项树B0: 
        41(0) is root
    2. 二项树B1: 
        28(1) is root
        33(0) is 28's child
    3. 二项树B2: 
         7(2) is root
        15(1) is  7's child
        25(0) is 15's child
        12(0) is 15's next

[link_source_code]: https://github.com/wangkuiwu/datastructs_and_algorithm/tree/master/source/heap/binomial/c
 
