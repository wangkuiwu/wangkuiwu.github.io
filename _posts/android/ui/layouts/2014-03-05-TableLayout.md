---
layout: post
title: "Android布局之 TableLayout"
description: "android layout"
category: android
tags: [android]
date: 2014-03-05 09:01
---

> 本文介绍TableLayout布局。

> 点击查看：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/layouts/TableLayout)

> **目录**  
[1. TableLayout简介](#anchor1)  
[2. TableLayout示例](#anchor2)  


<a name="anchor1"></a>
# 1. TableLayout简介

TableLayout是表格布局。TableLayout 可设置的属性包括全局属性及单元格属性。

## 1.1 全局属性

有以下3个参数：

|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| android:stretchColumns | 设置可伸展的列。该列可以向行方向伸展，最多可占据一整行。 |
| android:shrinkColumns | 设置可收缩的列。当该列子控件的内容太多，已经挤满所在行，那么该子控件的内容将往列方向显示。 |
| android:collapseColumns | 设置要隐藏的列。 |

 

示例：

    android:stretchColumns="0"     ----  第0列可伸展
    android:shrinkColumns="1,2"    ----  第1,2列皆可收缩
    android:collapseColumns="*"    ----  隐藏所有行

说明：列可以同时具备stretchColumns及shrinkColumns属性，若此，那么当该列的内容N多时，将“多行”显示其内容。（这里不是真正的多行，而是系统根据需要自动调节该行的layout_height）


## 1.2 单元格属性

有以下2个参数：

|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| android:layout_column | 指定该单元格在第几列显示 |
| android:layout_span | 指定该单元格占据的列数（未指定时，为1） |

 
示例：

    android:layout_column="1"   ----  该控件显示在第1列
    android:layout_span="2"     ----  该控件占据2列

说明：一个控件也可以同时具备这两个特性。



<a name="anchor2"></a>
# 2. TableLayout示例

下面通过两则示例来演示TableLayout的基本用法。

### layout布局

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:padding="3dip"
        >
        
        <!-- 第1个TableLayout，用于描述表中的列属性。第0列可伸展，第1列可收缩 ，第2列被隐藏-->
        <TextView 
            android:text="表1：全局设置：列属性设置"
            android:layout_height="wrap_content" 
            android:layout_width="wrap_content"
            android:textSize="15sp"
            android:background="#7f00ffff"/>
        <TableLayout                
            android:id="@+id/table1"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:stretchColumns="0"
            android:shrinkColumns="1"
            android:collapseColumns="2"
            android:padding="3dip">
            <TableRow>
                <Button android:text="该列可伸展"/>
                <Button android:text="该列可收缩"/>
                <Button android:text="我被隐藏了"/>
            </TableRow>
            
            <TableRow>
                <TextView android:text="我向行方向伸展，我可以很长"/>
                <TextView android:text="我向列方向收缩，我可以很深; 我向列方向收缩，我可以很深"/>
            </TableRow>        
            
        </TableLayout>
        
        <!-- 第2个TableLayout，用于描述表中单元格的属性，包括：android:layout_column 及android:layout_span-->
        <TextView 
            android:text="表2:单元格设置：指定单元格属性设置"
            android:layout_height="wrap_content" 
            android:layout_width="wrap_content"
            android:textSize="15sp"
            android:background="#7f00ffff"/>    
        <TableLayout
            android:id="@+id/table2"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:padding="3dip">
            <TableRow>
                <Button android:text="第0列"/>
                <Button android:text="第1列"/>
                <Button android:text="第2列"/>
            </TableRow>
            
            <TableRow>
                <TextView android:text="我被指定在第1列" android:layout_column="1"/>
            </TableRow>
                
            <TableRow>
                <TextView
                    android:text="我跨1到2列，不信你看！"
                    android:layout_column="1"
                    android:layout_span="2"
                    />
            </TableRow>
            
        </TableLayout>    
        
        <!-- 第3个TableLayout，使用可伸展特性布局-->
        <TextView 
            android:text="表3：应用一，非均匀布局"
            android:layout_height="wrap_content" 
            android:layout_width="wrap_content"
            android:textSize="15sp"
            android:background="#7f00ffff"/>
        <TableLayout
            android:id="@+id/table3"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:stretchColumns="*"
            android:padding="3dip"
            >
            <TableRow>
                <Button android:text="一" ></Button>
                <Button android:text="两字"></Button>
                <Button android:text="三个字" ></Button>
            </TableRow>
        </TableLayout>
        
        <!-- 第4个TableLayout，使用可伸展特性，并指定每个控件宽度一致，如1dip-->
        <TextView 
            android:text="表4：应用二，均匀布局"
            android:layout_height="wrap_content" 
            android:layout_width="wrap_content"
            android:textSize="15sp"
            android:background="#7f00ffff"/>
        <TableLayout
            android:id="@+id/table4"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:stretchColumns="*"
            android:padding="3dip"
            >
            <TableRow>
                <!-- 
                 -->
                <Button android:text="一" android:layout_width="1dip"></Button>
                <Button android:text="两字" android:layout_width="1dip"></Button>
                <Button android:text="三个字" android:layout_width="1dip"></Button>
            </TableRow>
        </TableLayout>
    </LinearLayout>

![img](/media/pic/android/layouts/tablelayout_01.jpg)

