---
layout: post
title: "Android控件篇05之 RadioButton"
description: "android training"
category: android
tags: [android]
date: 2014-02-05 09:11
---


> 本文介绍RadioButton控件。下面的内容都是参考"[RadioButton测试代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/RadioButton/RadioTest)"进行的讲解，感兴趣的可以进行测试验证。


<a name="anchor1"></a>
# 1. RadioButton基本使用

## 1.1 RadioButton的父类

下面是RadioButton的继承关系图

    java.lang.Object
      ↳ android.view.View
        ↳ android.widget.TextView
          ↳ android.widget.Button
            ↳ android.widget.CompoundButton
              ↳ android.widget.RadioButton

从中，可以看出Button和CompoundButton都是RadioButton父类。这也就意味着，RadioButton同时具有它们的特性。


## 1.2 RadioButton基本定义


    <RadioGroup
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical">
        
        <RadioButton android:id="@+id/radio_one"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:checked="true"
            android:onClick="onRadioButtonClicked"
            android:text="@string/one"/>
            
        <RadioButton android:id="@+id/radio_two"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:onClick="onRadioButtonClicked"
            android:text="@string/two"/>
        
        <RadioButton android:id="@+id/radio_three"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:onClick="onRadioButtonClicked"
            android:text="@string/three"/>
            
    </RadioGroup>



## 1.3 RadioButton事件的监听

RadioButton事件的监听，有两种方法：

**第一种**：实现View.OnClickListener接口。这个和监听Button的方式一样，它监听的是点击动作！

**第二种**：实现CompoundButton.OnCheckedChangeListener接口。这个和第一种方法的区别是，它是监听状态(监听按钮状态的变化)，而第一种方法是监听动作(监听按钮的点击事件)。

具体的实现如下：

    public class RadioTest extends Activity {

        private TextView mView;

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            mView = (TextView) findViewById(R.id.tv_intro);

            RadioButton mTwo = (RadioButton) findViewById(R.id.radio_two);
            mTwo.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
                @Override
                public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                    if (isChecked) {
                        mView.setText("two checked!");
                    } else {
                        mView.setText("two un-checked!");
                    }   
                }   
            }); 
     
        }   


        public void onRadioButtonClicked(View view) {
            boolean checked = ((RadioButton) view).isChecked();
                    
            switch(view.getId()) {
                case R.id.radio_one:
                    Toast.makeText(this, "one checked="+checked, Toast.LENGTH_SHORT).show();
                    break;
                case R.id.radio_two:
                    Toast.makeText(this, "two checked="+checked, Toast.LENGTH_SHORT).show();
                    break;
                case R.id.radio_three:
                    Toast.makeText(this, "three checked="+checked, Toast.LENGTH_SHORT).show();
                    break;
                default:
                    break;
            }   
        }
    }


