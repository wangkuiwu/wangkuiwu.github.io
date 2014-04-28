---
layout: post
title: "邻接矩阵无向图(一) 之C语言的实现"
description: "matrix undirected graph"
category: datastructure
tags: []
---


# 概要

本章介绍邻接矩阵无向图。在"[图的理论基础][link_graph_thesis]"中已经对图进行了理论介绍，这里就不再对图的概念进行重复说明了。和以往一样，本文会先给出C语言的实现；后续再分别给出C++和Java版本的实现。实现的语言虽不同，但是原理如出一辙，选择其中之一进行了解即可。若文章有错误或不足的地方，请不吝指出！ 

**目录**

+ [邻接矩阵无向图的介绍](#anchor1)
+ [邻接矩阵无向图的代码说明](#anchor2)
+ [邻接矩阵无向图的完整源码和测试程序](#anchor3)

-----------------
转载请注明出处：TODO[http://www.cnblogs.com/skywang12345/](http://www.cnblogs.com/skywang12345/)




<a id="anchor1"></a>
# 邻接矩阵无向图的介绍

邻接矩阵无向图是指通过邻接矩阵表示的无向图。

<img src="http://www.cnblogs.com/images/cnblogs_com/skywang12345/552879/t_f1.jpg" alt="TODO" width="40px"/ >

上面的图G1包含了"A,B,C,D,E,F,G"共7个顶点，而且包含了"(A,C),(A,D),(A,F),(B,C),(C,D),(E,G),(F,G)"共7条边。由于这是无向图，所以边(A,C)和边(C,A)是同一条边；这里列举边时，是按照字母先后顺序列举的。

上图右边的矩阵是G1在内存中的邻接矩阵示意图。A[i][j]=1表示第i个顶点与第j个顶点是邻接点，A[i][j]=0则表示它们不是邻接点；而A[i][j]表示的是第i行第j列的值；例如，A[1,2]=1，表示第1个顶点(即顶点B)和第2个顶点(C)是邻接点。




<a id="anchor2"></a>
# 邻接矩阵无向图的代码说明

## 1. 基本定义

    // 邻接矩阵
    typedef struct _graph
    {
        char vexs[MAX];       // 顶点集合
        int vexnum;           // 顶点数
        int edgnum;           // 边数
        int matrix[MAX][MAX]; // 邻接矩阵
    }Graph, *PGraph;


Graph是邻接矩阵对应的结构体。

vexs用于保存顶点，vexnum是顶点数，edgnum是边数；matrix则是用于保存矩阵信息的二维数组。例如，matrix[i][j]=1，则表示"顶点i(即vexs[i])"和"顶点j(即vexs[j])"是邻接点；matrix[i][j]=0，则表示它们不是邻接点。

## 2. 创建矩阵

这里介绍提供了两个创建矩阵的方法。一个是**用已知数据**，另一个则**需要用户手动输入数据**。

### 2.1 创建图(用已提供的矩阵)

    /*
     * 创建图(用已提供的矩阵)
     */
    Graph* create_example_graph()
    {
        char vexs[] = {'A', 'B', 'C', 'D', 'E', 'F', 'G'};
        char edges[][2] = {
            {'A', 'C'}, 
            {'A', 'D'}, 
            {'A', 'F'}, 
            {'B', 'C'}, 
            {'C', 'D'}, 
            {'E', 'G'}, 
            {'F', 'G'}}; 
        int vlen = LENGTH(vexs);
        int elen = LENGTH(edges);
        int i, p1, p2;
        Graph* pG;
        
        // 输入"顶点数"和"边数"
        if ((pG=(Graph*)malloc(sizeof(Graph))) == NULL )
            return NULL;
        memset(pG, 0, sizeof(Graph));

        // 初始化"顶点数"和"边数"
        pG->vexnum = vlen;
        pG->edgnum = elen;
        // 初始化"顶点"
        for (i = 0; i < pG->vexnum; i++)
        {
            pG->vexs[i] = vexs[i];
        }

        // 初始化"边"
        for (i = 0; i < pG->edgnum; i++)
        {
            // 读取边的起始顶点和结束顶点
            p1 = get_position(*pG, edges[i][0]);
            p2 = get_position(*pG, edges[i][1]);

            pG->matrix[p1][p2] = 1;
            pG->matrix[p2][p1] = 1;
        }

        return pG;
    }

create_example_graph是的作用是创建一个邻接矩阵无向图。

**注意：该方法创建该的无向图，就是上面图G1。**


### 2.2 创建图(自己输入)

    /*
     * 创建图(用已提供的矩阵)
     */
    Graph* create_example_graph()
    {
        char vexs[] = {'A', 'B', 'C', 'D', 'E', 'F', 'G'};
        char edges[][2] = {
            {'A', 'C'}, 
            {'A', 'D'}, 
            {'A', 'F'}, 
            {'B', 'C'}, 
            {'C', 'D'}, 
            {'E', 'G'}, 
            {'F', 'G'}}; 
        int vlen = LENGTH(vexs);
        int elen = LENGTH(edges);
        int i, p1, p2;
        Graph* pG;
        
        // 输入"顶点数"和"边数"
        if ((pG=(Graph*)malloc(sizeof(Graph))) == NULL )
            return NULL;
        memset(pG, 0, sizeof(Graph));

        // 初始化"顶点数"和"边数"
        pG->vexnum = vlen;
        pG->edgnum = elen;
        // 初始化"顶点"
        for (i = 0; i < pG->vexnum; i++)
        {
            pG->vexs[i] = vexs[i];
        }

        // 初始化"边"
        for (i = 0; i < pG->edgnum; i++)
        {
            // 读取边的起始顶点和结束顶点
            p1 = get_position(*pG, edges[i][0]);
            p2 = get_position(*pG, edges[i][1]);

            pG->matrix[p1][p2] = 1;
            pG->matrix[p2][p1] = 1;
        }

        return pG;
    }

create_graph()是读取用户的输入，将输入的数据转换成对应的无向图。




<a id="anchor3"></a>
# 邻接矩阵无向图的完整源码和测试程序

TODO[完整源码][link_source_code]

上面的源码中包含了测试代码，测试程序的运行结果如下：

    Martix Graph:
    0 0 1 1 0 1 0 
    0 0 1 0 0 0 0 
    1 1 0 1 0 0 0 
    1 0 1 0 0 0 0 
    0 0 0 0 0 0 1 
    1 0 0 0 0 0 1 
    0 0 0 0 1 1 0


[link_graph_thesis]: http://www.cnblogs.com/skywang12345/ "GraphThesis"
[link_source_code]: http://www.cnblogs.com/skywang12345/ "GraphThesis"
