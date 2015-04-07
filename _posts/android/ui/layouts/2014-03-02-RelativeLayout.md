---
layout: post
title: "Android布局之 RelativeLayout"
description: "android layout"
category: android
tags: [android]
date: 2014-03-02 09:01
---

> 本文介绍RelativeLayout布局。

> 点击查看：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/layouts/RelativeLayout)

> **目录**  
[1. RelativeLayout简介](#anchor1)  
[2. RelativeLayout示例](#anchor2)  


<a name="anchor1"></a>
# 1. RelativeLayout简介

RelativeLayout是相对布局，其中的控件位置都是相对而言的。

先看看RelativeLayout的相关属性，再通过示例来演示。

<br/>
## 1.1 相对于父控件的属性

|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| android:layout_alignParentStart | 表示widget的左边和Container的起始边缘对齐。 | 
| android:layout_alignParentEnd | 表示widget的左边和Container的结束边缘对齐。 | 
| android:layout_alignParentTop | 表示widget的顶部和Container的顶部对齐。 | 
| android:layout_alignParentBottom | 表示widget的底部和Container的底部对齐。 | 
| android:layout_alignParentLeft | 表示widget的左边和Container的左边对齐。 | 
| android:layout_alignParentRight | 表示widget的右边和Container的右边对齐。 | 
| android:layout_centerInParent | 表示widget处于Container平面上的正中间。 | 
| android:layout_centerHorizontal | 表示widget处于Container水平方向上的中间。 | 
| android:layout_centerVertical | 表示widget处于Container垂直方向上的中间。 | 
| android:layout_alignWithParentIfMissing | 若设置true，则当该控件layout_toLeftOf, layout_toRightOf等找不到相对的参考widget时，就以父container为参考 |



<br/>
## 1.2 相对于其他控件的属性

|             属性名称            |                          说明                       |
| ------------------------------- | --------------------------------------------------- |
| android:layout_alignStart | 表示该widget的起始边缘与参数值标识的widget的起始边缘对齐。 | 
| android:layout_alignEnd | 表示该widget的结束边缘与参数值标识的widget的结束边缘对齐。 | 
| android:layout_alignTop | 表示该widget的顶部参数值标识的widget的顶部对齐。 | 
| android:layout_alignBottom | 表示该widget的底部与参数值标识的widget的底部对齐。 | 
| android:layout_alignLeft | 表示该widget的左边与参数值标识的widget的左边对齐。 | 
| android:layout_alignRight | 表示该widget的右边参数值标识的widget的右边对齐。 | 
| android:layout_alignBaseline | 表示该widget的BaseLine与参数值标识的widget的BaseLine对齐。 | 
| android:layout_toStartOf | 表示该widget结束边缘与参数值标识的widget的起始边缘对齐 | 
| android:layout_toEndOf | 表示该widget起始边缘与参数值标识的widget的结束边缘对齐 | 
| android:layout_toLeftOf | 表示该widget位于参数值标识的widget的左方。 | 
| android:layout_toRightOf | 表示该widget位于参数值标识的widget的右方。 | 
| android:layout_above | 表示该widget位于参数值标识的widget的上方。 | 
| android:layout_below | 表示该widget位于参数值标识的widget的下方。 | 



<a name="anchor2"></a>
# 2. RelativeLayout示例

下面通过一则示例来演示RelativeLayout的基本用法。

## 示例说明

定义一个RelativeLayout，它的宽和高都是match_parent。RelativeLayout里面包含3个控件：  
(01) 包含一个EditText，在RelativeLayout中居中显示。  
(02) 包含一个"确定"按钮，在EditText下面，并与EditText右对齐。  
(03) 包含一个"取消"按钮，在"确定按钮"的左边，并与"确定按钮"顶部对齐。  

## layout布局

    <?xml version="1.0" encoding="utf-8"?>
    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent" >

        <EditText
            android:id="@+id/et"
            android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:layout_centerInParent="true"
            android:hint="please input your name:"
            android:textSize="16sp" />

        <Button
            android:id="@+id/ok"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_below="@id/et"
            android:layout_alignRight="@id/et"
            android:text="确定" />

        <Button
            android:id="@+id/cancel"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_toLeftOf="@id/ok"
            android:layout_alignTop="@id/ok"
            android:text="取消" />

    </RelativeLayout>

