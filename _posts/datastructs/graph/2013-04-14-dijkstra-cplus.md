---
layout: post
title: "Dijkstra算法(二)之 C++详解"
description: "dijkstra"
category: datastructure
tags: [graph,c++]
date: 2013-04-14 12:25
---


> 本章是迪杰斯特拉算法的C++实现。

> **目录**  
> **1**. [迪杰斯特拉算法介绍](#anchor1)  
> **2**. [迪杰斯特拉算法图解](#anchor2)  
> **3**. [迪杰斯特拉算法的代码说明](#anchor3)  
> **4**. [迪杰斯特拉算法的源码](#anchor4)  




<a name="anchor1"></a>
# 迪杰斯特拉算法介绍

迪杰斯特拉(Dijkstra)算法是典型最短路径算法，用于计算一个节点到其他节点的最短路径。  
它的主要特点是以起始点为中心向外层层扩展(广度优先搜索思想)，直到扩展到终点为止。


<br/>
**基本思想**  

 &nbsp;&nbsp;&nbsp;&nbsp; 通过Dijkstra计算图G中的最短路径时，需要指定起点s(即从顶点s开始计算)。  

 &nbsp;&nbsp;&nbsp;&nbsp; 此外，引进两个集合S和U。S的作用是记录已求出最短路径的顶点(以及相应的最短路径长度)，而U则是记录还未求出最短路径的顶点(以及该顶点到起点s的距离)。  

 &nbsp;&nbsp;&nbsp;&nbsp; 初始时，S中只有起点s；U中是除s之外的顶点，并且U中顶点的路径是"起点s到该顶点的路径"。然后，从U中找出路径最短的顶点，并将其加入到S中；接着，更新U中的顶点和顶点对应的路径。 然后，再从U中找出路径最短的顶点，并将其加入到S中；接着，更新U中的顶点和顶点对应的路径。 ... 重复该操作，直到遍历完所有顶点。



<br/>
**操作步骤**  

**(1)** 初始时，S只包含起点s；U包含除s外的其他顶点，且U中顶点的距离为"起点s到该顶点的距离"[例如，U中顶点v的距离为(s,v)的长度，然后s和v不相邻，则v的距离为∞]。  

**(2)** 从U中选出"距离最短的顶点k"，并将顶点k加入到S中；同时，从U中移除顶点k。  

**(3)** 更新U中各个顶点到起点s的距离。之所以更新U中顶点的距离，是由于上一步中确定了k是求出最短路径的顶点，从而可以利用k来更新其它顶点的距离；例如，(s,v)的距离可能大于(s,k)+(k,v)的距离。  

**(4)** 重复步骤(2)和(3)，直到遍历完所有顶点。  


单纯的看上面的理论可能比较难以理解，下面通过实例来对该算法进行说明。


<a name="anchor2"></a>
# 迪杰斯特拉算法图解

![img](/media/pic/datastruct_algrithm/graph/dijkstra/01.jpg)


以上图G4为例，来对迪杰斯特拉进行算法演示(以第4个顶点D为起点)。

![img](/media/pic/datastruct_algrithm/graph/dijkstra/02.jpg)


**初始状态**：S是已计算出最短路径的顶点集合，U是未计算除最短路径的顶点的集合！   
**第1步**：将顶点D加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0)}, U={A(∞),B(∞),C(3),E(4),F(∞),G(∞)}。
  &nbsp;&nbsp;&nbsp;&nbsp;注:C(3)表示C到起点D的距离是3。     

**第2步**：将顶点C加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;上一步操作之后，U中顶点C到起点D的距离最短；因此，将C加入到S中，同时更新U中顶点的距离。以顶点F为例，之前F到D的距离为∞；但是将C加入到S之后，F到D的距离为9=(F,C)+(C,D)。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0),C(3)}, U={A(∞),B(23),E(4),F(9),G(∞)}。  

**第3步**：将顶点E加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;上一步操作之后，U中顶点E到起点D的距离最短；因此，将E加入到S中，同时更新U中顶点的距离。还是以顶点F为例，之前F到D的距离为9；但是将E加入到S之后，F到D的距离为6=(F,E)+(E,D)。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0),C(3),E(4)}, U={A(∞),B(23),F(6),G(12)}。  

**第4步**：将顶点F加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0),C(3),E(4),F(6)}, U={A(22),B(13),G(12)}。

**第5步**：将顶点G加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0),C(3),E(4),F(6),G(12)}, U={A(22),B(13)}。

**第6步**：将顶点B加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0),C(3),E(4),F(6),G(12),B(13)}, U={A(22)}。

**第7步**：将顶点A加入到S中。  
  &nbsp;&nbsp;&nbsp;&nbsp;此时，S={D(0),C(3),E(4),F(6),G(12),B(13),A(22)}。

此时，起点D到各个顶点的最短距离就计算出来了：**A(22) B(13) C(3) D(0) E(4) F(6) G(12)**。




<a name="anchor3"></a>
# 迪杰斯特拉算法的代码说明

以"邻接矩阵"为例对迪杰斯特拉算法进行说明，对于"邻接表"实现的图在后面会给出相应的源码。

## 1. 基本定义



    class MatrixUDG {
        #define MAX    100
        #define INF    (~(0x1<<31))        // 无穷大(即0X7FFFFFFF)
        private:
            char mVexs[MAX];    // 顶点集合
            int mVexNum;             // 顶点数
            int mEdgNum;             // 边数
            int mMatrix[MAX][MAX];   // 邻接矩阵

        public:
            // 创建图(自己输入数据)
            MatrixUDG();
            // 创建图(用已提供的矩阵)
            //MatrixUDG(char vexs[], int vlen, char edges[][2], int elen);
            MatrixUDG(char vexs[], int vlen, int matrix[][9]);
            ~MatrixUDG();

            // 深度优先搜索遍历图
            void DFS();
            // 广度优先搜索（类似于树的层次遍历）
            void BFS();
            // prim最小生成树(从start开始生成最小生成树)
            void prim(int start);
            // 克鲁斯卡尔（Kruskal)最小生成树
            void kruskal();
            // Dijkstra最短路径
            void dijkstra(int vs, int vexs[], int dist[]);
            // 打印矩阵队列图
            void print();

        private:
            // 读取一个输入字符
            char readChar();
            // 返回ch在mMatrix矩阵中的位置
            int getPosition(char ch);
            // 返回顶点v的第一个邻接顶点的索引，失败则返回-1
            int firstVertex(int v);
            // 返回顶点v相对于w的下一个邻接顶点的索引，失败则返回-1
            int nextVertex(int v, int w);
            // 深度优先搜索遍历图的递归实现
            void DFS(int i, int *visited);
            // 获取图中的边
            EData* getEdges();
            // 对边按照权值大小进行排序(由小到大)
            void sortEdges(EData* edges, int elen);
            // 获取i的终点
            int getEnd(int vends[], int i);
    };

MatrixUDG是邻接矩阵对应的结构体。  
mVexs用于保存顶点，mVexNum是顶点数，mEdgNum是边数；mMatrix则是用于保存矩阵信息的二维数组。例如，mMatrix[i][j]=1，则表示"顶点i(即mVexs[i])"和"顶点j(即mVexs[j])"是邻接点；mMatrix[i][j]=0，则表示它们不是邻接点。

### 2. 迪杰斯特拉算法


    /*
     * Dijkstra最短路径。
     * 即，统计图中"顶点vs"到其它各个顶点的最短路径。
     *
     * 参数说明：
     *       vs -- 起始顶点(start vertex)。即计算"顶点vs"到其它顶点的最短路径。
     *     prev -- 前驱顶点数组。即，prev[i]的值是"顶点vs"到"顶点i"的最短路径所经历的全部顶点中，位于"顶点i"之前的那个顶点。
     *     dist -- 长度数组。即，dist[i]是"顶点vs"到"顶点i"的最短路径的长度。
     */
    void MatrixUDG::dijkstra(int vs, int prev[], int dist[])
    {
        int i,j,k;
        int min;
        int tmp;
        int flag[MAX];      // flag[i]=1表示"顶点vs"到"顶点i"的最短路径已成功获取。
        
        // 初始化
        for (i = 0; i < mVexNum; i++)
        {
            flag[i] = 0;              // 顶点i的最短路径还没获取到。
            prev[i] = 0;              // 顶点i的前驱顶点为0。
            dist[i] = mMatrix[vs][i]; // 顶点i的最短路径为"顶点vs"到"顶点i"的权。
        }

        // 对"顶点vs"自身进行初始化
        flag[vs] = 1;
        dist[vs] = 0;

        // 遍历mVexNum-1次；每次找出一个顶点的最短路径。
        for (i = 1; i < mVexNum; i++)
        {
            // 寻找当前最小的路径；
            // 即，在未获取最短路径的顶点中，找到离vs最近的顶点(k)。
            min = INF;
            for (j = 0; j < mVexNum; j++)
            {
                if (flag[j]==0 && dist[j]<min)
                {
                    min = dist[j];
                    k = j;
                }
            }
            // 标记"顶点k"为已经获取到最短路径
            flag[k] = 1;

            // 修正当前最短路径和前驱顶点
            // 即，当已经"顶点k的最短路径"之后，更新"未获取最短路径的顶点的最短路径和前驱顶点"。
            for (j = 0; j < mVexNum; j++)
            {
                tmp = (mMatrix[k][j]==INF ? INF : (min + mMatrix[k][j]));
                if (flag[j] == 0 && (tmp  < dist[j]) )
                {
                    dist[j] = tmp;
                    prev[j] = k;
                }
            }
        }

        // 打印dijkstra最短路径的结果
        cout << "dijkstra(" << mVexs[vs] << "): " << endl;
        for (i = 0; i < mVexNum; i++)
            cout << "  shortest(" << mVexs[vs] << ", " << mVexs[i] << ")=" << dist[i] << endl;
    }



<a name="anchor4"></a>
# 迪杰斯特拉算法的源码

这里分别给出"邻接矩阵图"和"邻接表图"的迪杰斯特拉算法源码。

**1**. [邻接矩阵源码(MatrixUDG.cpp)][link_source_code_01]  

**2**. [邻接表源码(ListUDG.cpp)][link_source_code_02]  


[link_source_code_01]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/graph/dijkstra/udg/cplus/MatrixUDG.cpp
[link_source_code_02]: https://github.com/wangkuiwu/datastructs_and_algorithm/blob/master/source/graph/dijkstra/udg/cplus/ListUDG.cpp
