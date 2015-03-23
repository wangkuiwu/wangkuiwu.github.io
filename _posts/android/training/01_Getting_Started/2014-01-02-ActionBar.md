---
layout: post
title: "Android培训(一)开始篇02之 添加ActionBar"
description: "android training"
category: android
tags: [android]
date: 2014-01-02 09:01
---


> 本文Android3.0及以上版本中ActionBar的基本使用方法。

> **目录**  
[1. ActionBar说明](#anchor1)  
[2. ActionBar中添加按钮](#anchor2)  
[3. ActionBar状态设置](#anchor3)  
[4. 覆盖ActionBar](#anchor4)  



<a name="anchor1"></a>
# 1. ActionBar说明


在Android3.0(**API level 11**)及以上版本，可以通过在manifest中使用主题"**[Theme.Holo][link_google_theme] 及 它的其它相关主题**"来添加ActionBar。


例如，在"[ActionBar示例一][link_actionbar_01_add]"中有添加以下属性：

    android:theme="@android:style/Theme.Holo.Light.DarkActionBar"


如果要在"Android 2.1及以上版本"添加ActionBar，则需要通过引入"Android Suppor Library"的方式，来使用ActionBar。





<a name="anchor2"></a>
# 2. ActionBar中添加按钮


点击查看：[示例工程][link_actionbar_project01]

## 2.1 添加方式

**第一步**：通过添加xml文件来指令ActionBar中的按钮。例如，添加`res/menu/main_activity_actions.xml`，内容如下：

    <menu xmlns:android="http://schemas.android.com/apk/res/android" >
        <!-- Search, should appear as action button -->
        <item android:id="@+id/action_search"
              android:icon="@drawable/ic_action_search"
              android:title="@string/action_search"
              android:showAsAction="ifRoom" />
        <!-- Settings, should always be in the overflow -->
        <item android:id="@+id/action_settings"
              android:title="@string/action_settings"
              android:showAsAction="never" />
    </menu>

说明：    
android:showAsAction=["ifRoom" \| "never" \| "withText" \| "always" \| "collapseActionView"]   
(01) ifRoom -- 如果有空间，则显示该按钮。  
(02) withText -- 显示文本。  
(03) never -- 不将该按钮显示在ActionBar上。  
(04) always -- 总是将该按钮显示在ActionBar上。  
(05) collapseActionView -- 将按钮组合到ActionBar中。  


**第二步**：重写onCreateOptionsMenu()方法，并在解析上一步的xml文件。

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu items for use in the action bar
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.main_activity_actions, menu);
        return super.onCreateOptionsMenu(menu);
    }



## 2.2 响应按钮

覆盖onOptionsItemSelected()方法，从而对相应的ActionBar按钮做出响应。

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle presses on the action bar items
        switch (item.getItemId()) {
            case R.id.action_search:
                openSearch();
                return true;
            case R.id.action_settings:
                openSettings();
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }
    }







<a name="anchor3"></a>
# 3. ActionBar状态设置

点击查看：[示例工程][link_actionbar_project02]

## 3.1 使用主题

可以使用不同的ActionBar主题：

Theme.Holo: ActionBar和Body都是黑色。  
Theme.Holo.Light: ActionBar和Body都是浅色。  
Theme.Holo.Light.DarkActionBar: ActionBar是黑色，Body是浅色。  



## 3.2 自定义主题 

可以自定义主题，并在主题中指明ActionBar的各个属性。


例如，自定义主题，主题对应的style(res/values/themes.xml)内容如下：


    <?xml version="1.0" encoding="utf-8"?>
    <resources>
        <!-- the theme applied to the application or activity -->
        <style name="CustomActionBarTheme"
               parent="@android:style/Theme.Holo">
            <item name="android:actionBarStyle">@style/MyActionBar</item>
            <item name="android:actionBarTabTextStyle">@style/MyActionBarTabText</item>
            <item name="android:actionMenuTextColor">@color/actionbar_text</item>
        </style>

        <!-- ActionBar styles -->
        <style name="MyActionBar"
               parent="@android:style/Widget.Holo.ActionBar">
            <item name="android:titleTextStyle">@style/MyActionBarTitleText</item>
        </style>

        <!-- ActionBar title text -->
        <style name="MyActionBarTitleText"
               parent="@android:style/TextAppearance.Holo.Widget.ActionBar.Title">
            <item name="android:textColor">@color/actionbar_text</item>
        </style>

        <!-- ActionBar tabs text styles -->
        <style name="MyActionBarTabText"
               parent="@android:style/Widget.Holo.ActionBar.TabText">
            <item name="android:textColor">@color/actionbar_text</item>
        </style>
    </resources>

上面的主题，自定义了ActionBar的文字颜色等内容。  
注意：上面的style的parent是android自带的样式！





<a name="anchor4"></a>
# 4. 覆盖ActionBar

点击查看：[示例工程][link_actionbar_project03]

当ActionBar覆盖在Activity之上时，需要注意布局！在布局中，将Activity距离顶部的距离设为ActionBar的高度。

    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:paddingTop="?android:attr/actionBarSize">
        ...
    </RelativeLayout>



[link_google_theme]: http://developer.android.com/intl/zh-cn/reference/android/R.style.html#Theme_Holo
[link_actionbar_01_add]: https://github.com/wangkuiwu/android_applets/blob/master/training/01_getting_started/02_action_bar/02_adding_actionbar/bar1/AndroidManifest.xml

[link_actionbar_project01]:  https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/02_action_bar/02_adding_actionbar
[link_actionbar_project02]:  https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/02_action_bar/03_customzation_bar/bar2
[link_actionbar_project03]:  https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/02_action_bar/04_overlay_bar
