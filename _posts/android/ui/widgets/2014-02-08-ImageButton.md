---
layout: post
title: "Android控件篇08之 ImageButton"
description: "android training"
category: android
tags: [android]
date: 2014-02-08 09:11
---


<a name="anchor1"></a>
# 1. ImageButton介绍

ImageButton是图片按钮，用户能自定义按钮的图片。

ImageButton的drawable state说明如下表

|     属性                     |               说明                 |
| ---------------------------- | ---------------------------------- |
| android:drawable             | 默认图片，等于一个drawable资源 |
| android:state_pressed        | 按下状态的图片 |
| android:state_focused        | 获得焦点状态的图片，比如用户选择了一ImageButton |
| android:state_hovered        | 光标悬停状态的图片，通常与focused state相同，它是4.0的新特性 |
| android:state_selected       | 选中状态的图片，它与focus state并不完全一样，如一个list view 被选中的时候，它里面的各个子组件可能通过方向键，被选中了。 |
| android:state_checkable      | 按钮能否check，值为true或false。如果这个项目要用于对象的可选择状态，那么就要设置为true。如果这个项目要用于不可选状态，那么就要设置为false。（它只用于一个对象在可选和不可选之间的转换）。 |
| android:state_checked        | 选中状态的图片 |
| android:state_enabled        | 使用状态(比如，按钮能被正常点击状态)的图片，能够接受触摸或者点击事件 |
| android:state_activated      | 按钮被激活状态的图片 |
| android:state_window_focused | 如果这个项目要用于应用程序窗口的有焦点状态（应用程序是在前台），那么就要设置为true，否者设置false。 |



<a name="anchor2"></a>
# 2. ImageButton示例

创建一个activity，包含一个ImageButton：按下和未按下状态，分别显示不同图片。

**应用层代码**

    package com.skywang.control;

    import android.os.Bundle;
    import android.app.Activity;
    import android.view.Menu;

    public class ImageButtonTest extends Activity {

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.image_button_test);
        }
       
    }

**layout文件**

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >

        <ImageButton
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:scaleType="fitXY"
            android:src="@drawable/bt_del" />

    </LinearLayout>

bt_del.xml内容如下：

    <?xml version="1.0" encoding="utf-8"?>
     <selector xmlns:android="http://schemas.android.com/apk/res/android">
         <!-- 按下状态 -->
         <item android:state_pressed="true"
               android:drawable="@drawable/bt_del_down" />
         <!-- 获取焦点状态 -->
         <item android:state_focused="true"
               android:drawable="@drawable/bt_del_up" />
         <!-- 默认状态 -->
         <item android:drawable="@drawable/bt_del_up" />
     </selector>

