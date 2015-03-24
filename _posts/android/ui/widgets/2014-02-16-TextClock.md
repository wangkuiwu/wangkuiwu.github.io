---
layout: post
title: "Android控件篇16之 TextClock"
description: "android training"
category: android
tags: [android]
date: 2014-02-16 09:11
---


<a name="anchor1"></a>
# 1. TextClock简介

关于时间的文本显示，Android提供了DigitalClock和TextClock。  
DigitalClock是Android第1版本发布，功能很简单，只显示时间；在Android4.2(对应API Level 17)中，Android新增了TextClock。TextClock的功能更加强大，它不仅能显示时间，还能显示日期；而且支持自定义格式。因此，推荐在Android4.2之后都使用TextClock。

TextClock设置格式显示格式通过以下方式：

**(01) 设置12时制格式**  
**属性** android:format12Hour  
**方法** setFormat12Hour(CharSequence)  
上面两种方式都可以。  
**(02) 设置24时制格式**  
**属性** android:format24Hour  
**方法** setFormat24Hour(CharSequence)  
上面两种方式都可以。

设置属性值示例(1970/04/06 3:23am)

    "MM/dd/yy h:mmaa" -> "04/06/70 3:23am"
    "MMM dd, yyyy h:mmaa" -> "Apr 6, 1970 3:23am"
    "MMMM dd, yyyy h:mmaa" -> "April 6, 1970 3:23am"
    "E, MMMM dd, yyyy h:mmaa" -> "Mon, April 6, 1970 3:23am&
    "EEEE, MMMM dd, yyyy h:mmaa" -> "Monday, April 6, 1970 3:23am"
    "'Noteworthy day: 'M/d/yy" -> "Noteworthy day: 4/6/70"



<a name="anchor2"></a>
# 2. Clock示例

创建一个activity，包含1个DigitalClock和TextClock，TextClock按照制定格式显示日期和时间。

应用层代码

    public class TextClockTest extends Activity {

        private TextClock mTextClock;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.text_clock_test);
            
            mTextClock = (TextClock)findViewById(R.id.my_tc);
            // 设置12时制显示格式
            mTextClock.setFormat12Hour("EEEE, MMMM dd, yyyy h:mmaa");
            // 设置24时制显示格式
            mTextClock.setFormat24Hour("yyyy-MM-dd hh:mm, EEEE");
        }
    }

**layout文件**

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >
        
        <DigitalClock 
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="#FF00FF00"
            android:textSize="24sp"
            />
        
        <TextClock
            android:id="@+id/my_tc" 
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:textColor="#FFFF0000"
            android:textSize="24sp"
            />

    </LinearLayout>


运行效果如下图

![img](/media/pic/android/widgets/textclock01.jpg)

