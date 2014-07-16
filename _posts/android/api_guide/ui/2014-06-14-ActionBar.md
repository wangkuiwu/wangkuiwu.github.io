---
layout: post
title: "Android API指南(二)ActionBar篇"
description: "android training"
category: android
tags: [android]
date: 2014-06-14 09:11
---


> 本文介绍ActionBar。主要包括：ActionBar的基本用法，如何隐藏ActionBar，ActionBar的向上导航功能，以及使用使用ActionBar的Tab。


<a name="anchor1"></a>
# ActionBar的基本用法

点击查看：[ActionBar的基本用法完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/actionbar/01_basic/BarTest)

ActionBar和Menu的基本用于差不多，毕竟Menu通常是依附于ActionBar之上的。所以，使用ActionBar也需要在manfest中添加相关主题。例如：

    android:theme="@android:style/Theme.Holo"

当然，也可以是"@android:style/Theme.Holo"的子主题。


## 1. 定义menu菜单

新建res/menu/menu.xml，内容如下：

    <menu xmlns:android="http://schemas.android.com/apk/res/android" >

        <!-- search -->
        <item android:id="@+id/search"
              android:icon="@drawable/ic_action_search"
              android:title="@string/search"
              android:showAsAction="ifRoom" />

        <!-- setting -->
        <item android:id="@+id/setting"
              android:title="@string/setting"
              android:showAsAction="ifRoom|withText" />

        <!-- share -->
        <item android:id="@+id/share"
              android:icon="@drawable/ic_action_share"
              android:title="@string/share"
              android:showAsAction="ifRoom" />

    </menu>



## 2. 覆盖onCreateOptionsMenu()

在onCreateOptionsMenu()中展开菜单。


    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu items for use in the action bar
        MenuInflater inflater = getMenuInflater();
        inflater.inflate(R.menu.menu, menu);
        return super.onCreateOptionsMenu(menu);
    }   


## 3. 响应菜单点击事件


    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case R.id.search:
                Toast.makeText(getApplicationContext(), "search", 0).show();
                return true;
            case R.id.setting:
                Toast.makeText(getApplicationContext(), "setting", 0).show();
                return true;
            case R.id.share:
                Toast.makeText(getApplicationContext(), "share", 0).show();
                return true;
            default:
                return super.onOptionsItemSelected(item);
        }   
    }   




<a name="anchor2"></a>
# 隐藏ActionBar

点击查看：[隐藏ActionBar的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/actionbar/02_hide/BarTest)

通常有两种方式隐藏ActionBar。

## 1. 通过主题来隐藏ActionBar

    <activity android:theme="@android:style/Theme.Holo.NoActionBar">

## 2. 通过代码来隐藏

    ActionBar actionBar = getActionBar();
    actionBar.hide();




<a name="anchor3"></a>
# ActionBar向上导航

点击查看：[ActionBar向上导航的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/actionbar/03_homeup/BarTest)

## 1. 显示向上导航


    ActionBar actionBar = getActionBar();
    actionBar.setDisplayHomeAsUpEnabled(true);


## 2. 向上导航处理

向上导航对应的id是"android.R.id.home"。

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            ...
            case android.R.id.home:
                Toast.makeText(getApplicationContext(), "home!", 0).show();
                this.finish();
                return true;
            ...
        }   
    }   






<a name="anchor4"></a>
# ActionBar的Tab

点击查看：[ActionBarMenu的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/actionbar/04_tab/BarTest)

## 1. 新建Tab所需要的Fragment

Fragment对应的类：

    public class FragA extends Fragment {

        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container, 
                Bundle savedInstanceState) {
            return inflater.inflate(R.layout.frag_a, container, false);
        }   
    }

Fragment对应的配置文件：

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >                                                                                                                                                                                                   

        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:hint="@string/intro_fragA" />                                                                                                                                                                                           
        <Button
            android:id="@+id/send"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="@string/send"
            android:onClick="sendMessage" />                                                                                                                                                                                                 
    </LinearLayout>



## 2. 新建Tab并进行相关设置

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);

        // 获取ActionBar
        ActionBar bar = getActionBar();
        // 设置ActionBar模式
        bar.setNavigationMode(ActionBar.NAVIGATION_MODE_TABS);

        // 创建Fragment
        Fragment fragmentA = new FragA();
        Fragment fragmentB = new FragB();
        Fragment fragmentC = new FragC();

        // 新建3个Tab，并将这三个TAB添加到ActionBar中。
        // (01) setText()是设置标题
        // (02) setIcon()是设置图标
        // (02) setTabListener()是设置Tab监听器
        Tab tabA = bar.newTab().setText("A Tab").setIcon(R.drawable.ic_action_call).setTabListener(new MyTabListener(fragmentA));
        Tab tabB = bar.newTab().setText("B Tab").setIcon(R.drawable.ic_action_mail).setTabListener(new MyTabListener(fragmentB));
        Tab tabC = bar.newTab().setText("C Tab").setIcon(R.drawable.ic_action_video).setTabListener(new MyTabListener(fragmentC));

        bar.addTab(tabA);
        bar.addTab(tabB);
        bar.addTab(tabC);
    }   

说明：上面新建了3个Tab，并将这3个Tab添加到ActionBar中。每一个Tab都对应一个Fragment，在点击Tab时，我们显示当前被点击Tab的Fragment。


    class MyTabListener implements TabListener{
        private Fragment fragment; 

        public MyTabListener(Fragment fragment){ 
            this.fragment=fragment; 
        }   
        @Override 
        public void onTabReselected(Tab tab, FragmentTransaction ft) {
        }   
        @Override 
        public void onTabSelected(Tab tab, FragmentTransaction ft) {
            if (ft!=null) {
                ft.add(R.id.frag, fragment);
            }   
        }   
        @Override
        public void onTabUnselected(Tab tab, FragmentTransaction ft) {
            if (ft!=null) {
                ft.remove(fragment);
            }   
        }   
    }   















