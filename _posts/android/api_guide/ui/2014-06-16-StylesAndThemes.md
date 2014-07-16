---
layout: post
title: "Android API指南(二)Styles和Themes篇"
description: "android training"
category: android
tags: [android]
date: 2014-06-16 09:11
---


> 本文介绍Style样式和主题。


<a name="anchor1"></a>
# Style样式

Style样式可以划分为两种：Android系统自带的Style 和 自定义的Style。

点击查看：[Style的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/style_theme/01_style)


## 1. Android系统自带的Style

该Style定义在frameworks/base/core/res/res/values/styles.xml中。例如，android:TextAppearance的定义如下：

    <style name="TextAppearance">
        <item name="android:textColor">?textColorPrimary</item>
        <item name="android:textColorHighlight">?textColorHighlight</item>
        <item name="android:textColorHint">?textColorHint</item>
        <item name="android:textColorLink">?textColorLink</item>
        <item name="android:textSize">16sp</item>
        <item name="android:textStyle">normal</item>
    </style>


使用方法方式如下：

    <TextView
        style="@android:style/TextAppearance"
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:text="@string/intro"
        />


## 2. 自定义的Style

通常我们会新建文件res/values/styles.xml，然后将自定义的Style放在该文件下。如下示例：

    <style name="MyButtonStyle01">
        <item name="android:layout_width">wrap_content</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:textColor">#00FF00</item>
        <item name="android:textSize">24sp</item>
    </style>

如果要指定自定义Style的父类，则需要定义parent属性。如下：

    <style name="GreenText" parent="@android:style/TextAppearance">
        <item name="android:textColor">#00FF00</item>
    </style>


定义好的Style的使用方式如下：

    <Button
        style="@style/MyButtonStyle01"
        android:text="@string/send" />






<a name="anchor2"></a>
# Theme主题

Theme主题也可以划分为两种：Android系统自带的Theme 和 自定义的Theme。

点击查看：[Theme的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/style_theme/02_theme)



## 1. Android系统自带的Theme

该Theme定义在frameworks/base/core/res/res/values/themes.xml中。使用方法如下：

        <activity android:name="ThemeTest"
                  android:theme="@android:style/Theme.Holo"
                  android:label="@string/app_name">

说明：上面的Activity就使用了系统自带的主题"Theme.Holo"。


## 2. 自定义Theme

自定义Theme的方式和自定义Style的方式类似。

    <style name="CustomTheme" parent="@android:style/Theme.Holo">
        <item name="android:windowBackground">@android:color/white</item>
    </style>

说明：上面是自定义的一个CustomTheme。它继承于Theme.Hole，并且在父类的基础上将背景设为白色。


自定义的Theme和自带的Theme使用方法类似：

        <activity android:name="CustomTheme"
                  android:theme="@style/CustomTheme" />



