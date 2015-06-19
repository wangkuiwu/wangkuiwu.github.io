---
layout: post
title: "Android网络之理论篇(一) 网络层次"
description: "android network"
category: android
tags: [android]
date: 2015-05-01 09:01
---

> 本文介绍OSI的7层网络体系。

> **目录**  
[1. 网络层次概述](#anchor1)  
[2. 网络层次详解](#anchor2)  


<a name="anchor1"></a>
# 1. 网络层次概述

OSI（Open System Interconnection，开放系统互连）七层网络模型称为开放式系统互联参考模型。7层网络模型图如下图所示：  
(物链网传会表应)

![img](/media/pic/android/network/thesis/network_osi_7_01.jpg)

有时，可以将"7层模型"简化为"5层模型"。简化后的模型图如下：

![img](/media/pic/android/network/thesis/network_osi_7_02.jpg)

下面一张图是为了便于我们理解OSI 7层模型的一个比喻。建议在了解下面介绍的各个层次之后，再来仔细研究该图。

![img](/media/pic/android/network/thesis/network_osi_7_03.jpg)


<a name="anchor2"></a>
# 2. 网络层次详解

<a name="anchor2_1"></a>
## 2.1 物理层

物理层是OSI 7层模型中最低的一个层次。物理层包括物理传输媒介(如电缆连线、连接器)以及"传输介质的特性"。

物理层的数据单位称为比特(bit)。  
物理传输媒介包括：电话线、同轴电缆、双绞线、光导纤维电缆(即光纤)、无线与卫星通信信道。  
特性包括：物理特性和传输特性。物理特性包括“物理结构、形态尺寸、覆盖范围、价格等”，传输特性包含“带宽、传输速率、误码率、抗干扰能力等”。


<a name="anchor2_2"></a>
## 2.2 链路层

链路层控制着网络层与物理层之间的通信。它的主要功能是如何在不可靠的物理线路上进行数据的可靠传递。  
为了保证传输，从网络层接收到的数据被分割成特定的可被物理层传输的帧。帧是用来移动数据的结构包，它不仅包括原始数据，还包括发送方和接收方的物理地址以及检错和控制信息。其中的地址确定了帧将发送到何处，而纠错和控制信息则确保帧无差错到达。 如果在传送数据时，接收点检测到所传数据中有差错，就要通知发送方重发这一帧。

链路层的数据单位称是帧(frame)。  
链路层在不可靠的物理介质上提供可靠的传输。该层的作用包括：物理地址寻址、数据的成帧、流量控制、数据的检错、重发等。  
该层的协议的代表包括：HDLC(High-Level Data Link Control，即"高级数据链路控制")规程、PPP(Point to Point Protocol，即"点对点协议")。PPP是个协议族，它包含"链路控制协议LCP, IP控制协议IPCP, 口令授权协议PAP, 询问握手授权协议CHAP"这些协议。


<a name="anchor2_3"></a>
## 2.3 网络层

网络层负责在源和终点之间建立连接。它一般包括路由选择、流量控制、错误检查、网际连接等，其核心是网络连接服务。  
相同MAC标准的不同网段之间的数据传输一般只涉及到数据链路层，而不同的MAC标准之间的数据传输都涉及到网络层。例如IP路由器工作在网络层，因而可以实现多种网络间的互联。

网络层的数据单位称是数据包(packet)，包含了IP等信息。  
该层的协议包括：IP(Internet Protocol)协议、ARP(Address  Resolution Protocol)协议。


<a name="anchor2_4"></a>
## 2.4 传输层
传输层的作用是提供端到端的可靠的传输服务。

传输层的数据单位也是作数据包(packets)。但是，当你谈论TCP或UDP等具体的协议时又有特殊的叫法；例如，TCP的数据单元称为段(segments)，而UDP协议的数据单元称为“数据报(datagrams)”。  
传输层的协议包括：TCP(Transmission Control Protocol，即"传输控制协议")、UDP(User Datagram Protocol，即"用户数据报协议")、SPX(Sequenced Packet Exchange protocol，即"序列分组交换协议")等。


<a name="anchor2_5"></a>
## 2.5 会话层

会话层不参与具体的传输，它提供包括访问验证和会话管理在内的建立和维护应用之间通信的机制。

会话层的协议包括：RPC(Remote Procedure Call Protocol，即"远程过程调用协议")，SQL(Structured Query Language，即"结构化查询语言")等。


<a name="anchor2_6"></a>
## 2.6 表示层

表示层主要解决拥护信息的语法表示问题。她定义了数据格式及加密方式等内容。即，数据的压缩和解压缩，加密和解密等工作都由表示层负责。



<a name="anchor2_7"></a>
## 2.7 应用层

应用层为操作系统或网络应用程序提供访问网络服务的接口。

应用层的协议包括：Telnet、FTP(File Transfer Protocol，即"文件传输协议")、HTTP(HyperText Transfer Protocol，即"超文本传输协议")等。

