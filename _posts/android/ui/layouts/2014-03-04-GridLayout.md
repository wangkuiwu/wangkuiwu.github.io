---
layout: post
title: "Android布局之 GridLayout"
description: "android layout"
category: android
tags: [android]
date: 2014-03-04 09:01
---

> 本文介绍GridLayout布局。

> 点击查看：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/layouts/GridLayout)

> **目录**  
[1. GridLayout简介](#anchor1)  
[2. GridLayout示例](#anchor2)  


<a name="anchor1"></a>
# 1. GridLayout简介

GridLayout是Android4.0新提供的网格矩阵形式的布局控件。


## 1.1 GridLayout根视图的属性

所谓GridLayout根视图的属性，是指定义在GridLayout自身当中。后面还会介绍GridLayout所包含的子视图的属性，它指的是GridLayout中所包含的子视图元素的属性。

GridLayout的根视图属性如下：

|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| android:alignmentMode | 当设置alignMargins，使视图的外边界之间进行校准。可以取以下值：<br/> alignBounds -- 对齐子视图边界。<br/> alignMargins -- 对齐子视图边距。 |
| android:columnCount | GridLayout的最大列数 |
| android:rowCount | GridLayout的最大行数 |
| android:columnOrderPreserved | 当设置为true，使列边界显示的顺序和列索引的顺序相同。默认是true。 |
| android:orientation | GridLayout中子元素的布局方向。有以下取值：<br/> horizontal -- 水平布局。<br/> vertical -- 竖直布局。 |
| android:rowOrderPreserved | 当设置为true，使行边界显示的顺序和行索引的顺序相同。默认是true。 |
| android:useDefaultMargins | 当设置ture，当没有指定视图的布局参数时，告诉GridLayout使用默认的边距。默认值是false。 |


## 1.1 GridLayout子视图的属性

GridLayout子视图属性，是指GridLayout中所包含的子视图的可以定义的属性。


### 1.1.1 android:layout_column

属性说明： 显示该空间的列。例如，android:layout_column="0"，表示在第1列显示该控件；android:layout_column="1"，表示在第2列显示该控件。

layout文件示例：

    <?xml version="1.0" encoding="utf-8"?>
    <GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:rowCount="2"
        android:columnCount="3" >
    　　<Button
            android:id="@+id/one"
            android:layout_column="1"
            android:text="1"/>
    　　<Button
            android:id="@+id/two"
            android:layout_column="0"
            android:text="2"/>
    　　　<Button
            android:id="@+id/three"
            android:text="3"/>
    　　<Button
            android:id="@+id/devide"
            android:text="/"/>

    </GridLayout>

![img](/media/pic/android/layouts/gridlayout_01.jpg)

layout文件说明：

    android:orientation="horizontal" -- GridLayout中控件的布局方向是水平布局。
    android:rowCount="2"               -- GridLayout最大的行数为2行。
    android:columnCount="3"          -- GridLayout最大的列数为3列。
    android:layout_column="1"        -- 定义控件one的位于第2列。
    android:layout_column="0"        -- 定义该控two件的位于第1列。

### 1.1.2 android:layout_columnSpan

属性说明： 该控件所占的列数。例如，android:layout_columnSpan="2"，表示该控件占2列。

layout文件示例：

    <?xml version="1.0" encoding="utf-8"?>
    <GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:rowCount="2"
        android:columnCount="3" >
    　　<Button
            android:id="@+id/one"
            android:layout_column="0"
            android:layout_columnSpan="2"
            android:text="1"/>
    　　<Button
            android:id="@+id/two"
            android:text="2"/>
    　　　<Button
            android:id="@+id/three"
            android:text="3"/>
    　　<Button
            android:id="@+id/devide"
            android:text="/"/>

    </GridLayout>

![img](/media/pic/android/layouts/gridlayout_02.jpg)

layout文件说明：

  数字"1"实际上占据的空间大小是2列，但是第2列显示为空白。


### 1.1.3 android:layout_row

属性说明： 该控件所在行。  
例如，android:layout_row="0"，表示在第1行显示该控件；android:layout_row="1"，表示在第2行显示该控件。它和 android:layout_column类似。

 

### 1.1.4 android:layout_rowSpan

属性说明： 该控件所占的行数。  
例如，android:layout_rowSpan="2"，表示该控件占2行。它和 android:layout_columnSpan类似。

 

### 1.1.5 android:layout_gravity

属性说明：该控件的布局方式。布局方式的取值如下表：

|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| top | 控件置于容器顶部，不改变控件的大小。 |
| bottom | 控件置于容器底部，不改变控件的大小。 |
| left | 控件置于容器左边，不改变控件的大小。 |
| right | 控件置于容器右边，不改变控件的大小。 |
| center_vertical | 控件置于容器竖直方向中间，不改变控件的大小。 |
| fill_vertical | 如果需要，则往竖直方向延伸该控件。 |
| center_horizontal | 控件置于容器水平方向中间，不改变控件的大小。 |
| fill_horizontal | 如果需要，则往水平方向延伸该控件。 |
| center | 控件置于容器中间，不改变控件的大小。 |
| fill | 如果需要，则往水平、竖直方向延伸该控件。 |
| clip_vertical | 垂直剪切，剪切的方向基于该控件的top/bottom布局属性。若该控件的gravity是竖直的：若它的gravity是top的话，则剪切该控件的底部；若该控件的gravity是bottom的，则剪切该控件的顶部。 |
| clip_horizontal | 水平剪切，剪切的方向基于该控件的left/right布局属性。若该控件的gravity是水平的：若它的gravity是left的话，则剪切该控件的右边；若该控件的gravity是  right的，则剪切该控件的左边。 |
| start | 控件置于容器的起始处，不改变控件的大小。 |
| end | 控件置于容器的结束处，不改变控件的大小。 |



layout文件示例：

    <?xml version="1.0" encoding="utf-8"?>
    <GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:rowCount="2"
        android:columnCount="3" >
    　　<Button
            android:id="@+id/one"
            android:layout_column="0"
            android:layout_columnSpan="2"
            android:layout_gravity="fill"
            android:text="1"/>
    　　<Button
            android:id="@+id/two"
            android:text="2"/>
    　　　<Button
            android:id="@+id/three"
            android:text="3"/>
    　　<Button
            android:id="@+id/devide"
            android:text="/"/>

    </GridLayout>


对应的显示效果图：

![img](/media/pic/android/layouts/gridlayout_03.jpg)

<a name="anchor2"></a>
# 2. GridLayout示例

下面通过一则示例来演示GridLayout的基本用法。

### 示例说明

定义一个GridLayout，使它的布局类似于计算器布局。

### layout布局

    <?xml version="1.0" encoding="utf-8"?>
    <!-- GridLayout： 5行 4列 水平布局 -->
    <GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:rowCount="5"
        android:columnCount="4" >
    　　<Button
            android:id="@+id/one"
            android:text="1"/>
    　　<Button
            android:id="@+id/two"
            android:text="2"/>
    　　　<Button
            android:id="@+id/three"
            android:text="3"/>
    　　<Button
            android:id="@+id/devide"
            android:text="/"/>
    　　<Button
            android:id="@+id/four"
            android:text="4"/>
    　　<Button
            android:id="@+id/five"
            android:text="5"/>
    　　<Button
            android:id="@+id/six"
            android:text="6"/>
    　　<Button
            android:id="@+id/multiply"
            android:text="×"/>
    　　<Button
            android:id="@+id/seven"
            android:text="7"/>
    　　<Button
            android:id="@+id/eight"
            android:text="8"/>
    　　<Button
            android:id="@+id/nine"
            android:text="9"/>
        <Button
            android:id="@+id/minus"
            android:text="-"/>
        <Button
            android:id="@+id/zero"
            android:layout_columnSpan="2"
            android:layout_gravity="fill"
            android:text="0"/>
    　　<Button
            android:id="@+id/point"
            android:text="."/>
        <Button
            android:id="@+id/plus"
            android:layout_rowSpan="2"
            android:layout_gravity="fill"
            android:text="+"/>
        <Button
            android:id="@+id/equal"
            android:layout_columnSpan="3"
            android:layout_gravity="fill"
            android:text="="/> 
    </GridLayout>

![img](/media/pic/android/layouts/gridlayout_04.jpg)
