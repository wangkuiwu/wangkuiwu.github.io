---
layout: post
title: "Android控件篇06之 ToggleButton"
description: "android training"
category: android
tags: [android]
date: 2014-02-06 09:11
---


> 本文介绍ToggleButton控件。下面的内容都是参考"[ToggleButton测试代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/ToggleButton/ToggleTest)"进行的讲解，感兴趣的可以进行测试验证。


<a name="anchor1"></a>
# 1. ToggleButton基本使用

## 1.1 ToggleButton的父类

下面是ToggleButton的继承关系图

    java.lang.Object
      ↳ android.view.View
        ↳ android.widget.TextView
          ↳ android.widget.Button
            ↳ android.widget.CompoundButton
              ↳ android.widget.ToggleButton

从中，可以看出Button和CompoundButton都是ToggleButton父类。这也就意味着，ToggleButton同时具有它们的特性。


## 1.2 ToggleButton基本定义


    <ToggleButton 
        android:id="@+id/tb_simple"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textOn="@string/on"
        android:textOff="@string/off"
        android:onClick="onToggleClicked"/>

下面是自定义背景图的ToggleButton。

    <ToggleButton  
        android:id="@+id/tb_sync"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:background="@drawable/bg_toggle"
        android:textOn="@null"
        android:textOff="@null"
        android:disabledAlpha="1.5"
        android:gravity="center"
        android:onClick="onToggleClicked"
        />


## 1.3 ToggleButton事件的监听

ToggleButton事件的监听，有两种方法：

**第一种**：实现View.OnClickListener接口。这个和监听Button的方式一样，它监听的是点击动作！

**第二种**：实现CompoundButton.OnCheckedChangeListener接口。这个和第一种方法的区别是，它是监听状态(监听按钮状态的变化)，而第一种方法是监听动作(监听按钮的点击事件)。

下面是两种方法的实现：

    public class ToggleTest extends Activity {

        private TextView mView;

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            mView = (TextView) findViewById(R.id.tv_intro);

            ToggleButton mSync = (ToggleButton) findViewById(R.id.tb_sync);
            mSync.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
                @Override
                public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                    if (isChecked) {
                        mView.setText("checked!");
                    } else {
                        mView.setText("un-checked!");
                    }   
                }   
            }); 
        }   

        public void onToggleClicked(View view) {
            boolean checked = ((ToggleButton) view).isChecked();
             
            switch(view.getId()) {
                case R.id.tb_simple:
                    Toast.makeText(this, "simple checked="+checked, Toast.LENGTH_SHORT).show();
                    break;
                case R.id.tb_sync:
                    Toast.makeText(this, "sync checked="+checked, Toast.LENGTH_SHORT).show();
                    break;
                default:
                    break;
            }   
        }   
    }

