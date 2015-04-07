---
layout: post
title: "Android布局之 LinearLayout"
description: "android layout"
category: android
tags: [android]
date: 2014-03-01 09:01
---

> 本文介绍LinearLayout布局。

> 点击查看：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/layouts/LinearLayout)

> **目录**  
[1. LinearLayout基本用法](#anchor1)  
[2. LinearLayout中weight和weightSum的使用](#anchor2)  


<a name="anchor1"></a>
# 1. LinearLayout基本用法

LinearLayout是线性布局。其中的视图都是线性排队。

下面通过一则示例演示LinearLayout的基本用法。

## 示例说明

在程序中包含两组LinearLayout布局，每组LinearLayout布局都是水平(horizontal)排列，且每组都包含3个TextView。  
不同的是，第一组TextView的宽度都是wrap_content；而第2组TextView的宽度都是"LinearLayout布局总宽度的1/3"。

## layout布局

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent" >

        <!-- 第1组：水平方向上包含3个TextView，
             每个TextView的长度都是wrap_content -->
        <LinearLayout
            android:orientation="horizontal"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content" >
            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:gravity="center"
                android:textColor="#ff0000"
                android:text="ONE" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:gravity="center"
                android:textColor="#ff0000"
                android:text="TWO" />

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:gravity="center"
                android:textColor="#ff0000"
                android:text="THREE" />
        </LinearLayout>


        <!-- 第2组：水平方向上包含3个TextView，
             每个TextView的长度都是"LinearLayout长度的1/3" -->
        <LinearLayout
            android:orientation="horizontal"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content" >
            <TextView
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:gravity="center"
                android:textColor="#0000ff"
                android:text="ONE" />

            <TextView
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:gravity="center"
                android:textColor="#0000ff"
                android:text="TWO" />

            <TextView
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:gravity="center"
                android:textColor="#0000ff"
                android:text="THREE" />
        </LinearLayout>

    </LinearLayout>




<a name="anchor2"></a>
# 2. LinearLayout中weight和weightSum的使用

下面通过一则示例演示LinearLayout中weight和weightSum的使用。

## 示例说明

在LinearLayout中显示一个TextView，使用TextView的宽度是LinearLayout宽度的1/2，并且TextView水平居中。


## layout布局

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="horizontal"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:weightSum="4" >

        <TextView
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:background="@android:color/transparent" />

        <TextView
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="2"
            android:background="#666666"
            android:gravity="center"
            android:textColor="#0000ff"
            android:text="HOR_CENTER" />

    </LinearLayout>


