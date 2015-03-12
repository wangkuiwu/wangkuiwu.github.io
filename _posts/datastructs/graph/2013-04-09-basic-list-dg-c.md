---
layout: post
title: "邻接表有向图(一) 之C语言详解"
description: "list directed graph"
category: datastructure
tags: [graph,c]
date: 2013-04-09 09:25
---

> 本章介绍邻接表有向图。在"[图的理论基础][link_graph_thesis_todo]"中已经对图进行了理论介绍，这里就不再对图的概念进行重复说明了。和以往一样，本文会先给出C语言的实现；后续再分别给出C++和Java版本的实现。实现的语言虽不同，但是原理如出一辙，选择其中之一进行了解即可。若文章有错误或不足的地方，请不吝指出！ 

> **目录**  
> **1**. [邻接表有向图的介绍](#anchor1)  
> **2**. [邻接表有向图的代码说明](#anchor2)  
> **3**. [邻接表有向图的完整源码](#anchor3)  



<a name="anchor1"></a>
# 邻接表有向图的介绍

邻接表有向图是指通过邻接表表示的有向图。


![img](/media/pic/datastruct_algrithm/graph/basic/08.jpg)

上面的图G2包含了"A,B,C,D,E,F,G"共7个顶点，而且包含了"<A,B>,<B,C>,<B,E>,<B,F>,<C,E>,<D,C>,<E,B>,<E,D>,<F,G>"共9条边。

上图右边的矩阵是G2在内存中的邻接表示意图。每一个顶点都包含一条链表，该链表记录了"该顶点所对应的出边的另一个顶点的序号"。例如，第1个顶点(顶点B)包含的链表所包含的节点的数据分别是"2,4,5"；而这"2,4,5"分别对应"C,E,F"的序号，"C,E,F"都属于B的出边的另一个顶点。


<a name="anchor2"></a>
# 邻接表有向图的代码说明

## 1. 基本定义

    // 邻接表中表对应的链表的顶点
    typedef struct _ENode
    {
        int ivex;                   // 该边所指向的顶点的位置
        struct _ENode *next_edge;   // 指向下一条弧的指针
    }ENode, *PENode;

    // 邻接表中表的顶点
    typedef struct _VNode
    {
        char data;              // 顶点信息
        ENode *first_edge;      // 指向第一条依附该顶点的弧
    }VNode;

    // 邻接表
    typedef struct _LGraph
    {
        int vexnum;             // 图的顶点的数目
        int edgnum;             // 图的边的数目
        VNode vexs[MAX];
    }LGraph;


**(01)** LGraph是邻接表对应的结构体。 vexnum是顶点数，edgnum是边数；vexs则是保存顶点信息的一维数组。  
**(02)** VNode是邻接表顶点对应的结构体。 data是顶点所包含的数据，而first_edge是该顶点所包含链表的表头指针。  
**(03)** ENode是邻接表顶点所包含的链表的节点对应的结构体。 ivex是该节点所对应的顶点在vexs中的索引，而next_edge是指向下一个节点的。


## 2. 创建矩阵

这里介绍提供了两个创建矩阵的方法。一个是**用已知数据**，另一个则需要**用户手动输入数据**。

### 2.1 创建图(用已提供的矩阵)

    /*
     * 创建邻接表对应的图(用已提供的数据)
     */
    LGraph* create_example_lgraph()
    {
        char c1, c2;
        char vexs[] = {'A', 'B', 'C', 'D', 'E', 'F', 'G'};
        char edges[][2] = {
            {'A', 'B'}, 
            {'B', 'C'}, 
            {'B', 'E'}, 
            {'B', 'F'}, 
            {'C', 'E'}, 
            {'D', 'C'}, 
            {'E', 'B'}, 
            {'E', 'D'}, 
            {'F', 'G'}}; 
        int vlen = LENGTH(vexs);
        int elen = LENGTH(edges);
        int i, p1, p2;
        ENode *node1, *node2;
        LGraph* pG;


        if ((pG=(LGraph*)malloc(sizeof(LGraph))) == NULL )
            return NULL;
        memset(pG, 0, sizeof(LGraph));

        // 初始化"顶点数"和"边数"
        pG->vexnum = vlen;
        pG->edgnum = elen;
        // 初始化"邻接表"的顶点
        for(i=0; i<pG->vexnum; i++)
        {
            pG->vexs[i].data = vexs[i];
            pG->vexs[i].first_edge = NULL;
        }

        // 初始化"邻接表"的边
        for(i=0; i<pG->edgnum; i++)
        {
            // 读取边的起始顶点和结束顶点
            c1 = edges[i][0];
            c2 = edges[i][1];

            p1 = get_position(*pG, c1);
            p2 = get_position(*pG, c2);
            // 初始化node1
            node1 = (ENode*)malloc(sizeof(ENode));
            node1->ivex = p2;
            // 将node1链接到"p1所在链表的末尾"
            if(pG->vexs[p1].first_edge == NULL)
              pG->vexs[p1].first_edge = node1;
            else
                link_last(pG->vexs[p1].first_edge, node1);
        }

        return pG;
    }

该函数的作用是创建一个邻接表有向图。实际上，该方法创建的有向图，就是上面的图G2。


### 2.2 创建图(自己输入)

    /*
     * 创建邻接表对应的图(自己输入)
     */
    LGraph* create_lgraph()
    {
        char c1, c2;
        int v, e;
        int i, p1, p2;
        ENode *node1, *node2;
        LGraph* pG;

        // 输入"顶点数"和"边数"
        printf("input vertex number: ");
        scanf("%d", &v);
        printf("input edge number: ");
        scanf("%d", &e);
        if ( v < 1 || e < 1 || (e > (v * (v-1))))
        {
            printf("input error: invalid parameters!\n");
            return NULL;
        }
     
        if ((pG=(LGraph*)malloc(sizeof(LGraph))) == NULL )
            return NULL;
        memset(pG, 0, sizeof(LGraph));

        // 初始化"顶点数"和"边数"
        pG->vexnum = v;
        pG->edgnum = e;
        // 初始化"邻接表"的顶点
        for(i=0; i<pG->vexnum; i++)
        {
            printf("vertex(%d): ", i);
            pG->vexs[i].data = read_char();
            pG->vexs[i].first_edge = NULL;
        }

        // 初始化"邻接表"的边
        for(i=0; i<pG->edgnum; i++)
        {
            // 读取边的起始顶点和结束顶点
            printf("edge(%d): ", i);
            c1 = read_char();
            c2 = read_char();

            p1 = get_position(*pG, c1);
            p2 = get_position(*pG, c2);
            // 初始化node1
            node1 = (ENode*)malloc(sizeof(ENode));
            node1->ivex             = p2;
            // 将node1链接到"p1所在链表的末尾"
            if(pG->vexs[p1].first_edge == NULL)
              pG->vexs[p1].first_edge = node1;
            else
                link_last(pG->vexs[p1].first_edge, node1);
        }

        return pG;
    }

create_lgraph()是读取用户的输入，将输入的数据转换成对应的有向图。


<a name="anchor3"></a>
# 邻接表有向图的完整源码

点击查看：[源代码][link_source_code]


[link_graph_thesis_todo]: /2013/04/05/graph-thesis/
[link_source_code]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/graph/basic/dg/c/list_dg.c
