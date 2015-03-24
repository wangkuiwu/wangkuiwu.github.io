---
layout: post
title: "Android控件篇11之 ZoomButton"
description: "android training"
category: android
tags: [android]
date: 2014-02-11 09:11
---




<a name="anchor1"></a>
# 1 ZoomButton简介

ZoomButton，称为放大按钮。

实际上它继承于ImageButton，并在ImageButton基础上增加了“按下ZoomButton时，会不断上报点击事件”。至于上报的时间间隔，可以通过setZoomSpeed()去设置。



<a name="anchor2"></a>
# 2. ZoomButton示例

对比ZoomButton和ImageButton。写一个activity，包含一个ZoomButton和一个ImageButton。  
点击ZoomButton和ImageButton时，分别会放大不同的文本。测试时，请分别按住它们不放，查看效果。

**应用层代码**

    public class ZoomTest extends Activity implements View.OnClickListener {

        // ZoomButton
        private static int pauseSize= 12;
        private TextView mTvShow;     
        private ZoomButton mZoomPause;

        // ImageButton    
        private static int pauseCom= 12;
        private TextView mTvCom;
        private ImageButton mBtnCom;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.zoom_test);
            
            mTvShow = (TextView)findViewById(R.id.tv_show);        
            mZoomPause = (ZoomButton)findViewById(R.id.zoom_pause);
            mZoomPause.setOnClickListener(this);
        
            mTvCom = (TextView)findViewById(R.id.tv_com);
            mBtnCom = (ImageButton)findViewById(R.id.btn_com);
            mBtnCom.setOnClickListener(this);
        }

        public void onClick(View v) {
            switch(v.getId()) {
                case R.id.zoom_pause:{
                    // 按住ZoomButton不方，会不断的执行下面的操作：将文本放大
                    pauseSize += 2;
                    mTvShow.setTextSize(pauseSize);
                    break;
                }
                case R.id.btn_com:{
                    // 按住ImageButton，只有松开时才会执行下面操作：将文本放大
                    pauseCom += 2;
                    mTvCom.setTextSize(pauseCom);
                    break;
                }
                default:
                    break;
            }
        }
        
    }

**layout代码**


    <ZoomButton
        android:id="@+id/zoom_pause"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:scaleType="fitXY"
        android:background="@android:color/transparent"
        android:src="@drawable/btn_pause"/>
    
    <TextView
        android:id="@+id/tv_com"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="14sp"
        android:textColor="#FF0000AA"
        android:text="@string/text_com" />

    <ImageButton
        android:id="@+id/btn_com"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:scaleType="fitXY"
        android:background="@android:color/transparent"
        android:src="@drawable/btn_pause"/>


其中，btn_pause.xml文件内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <selector xmlns:android="http://schemas.android.com/apk/res/android" >
        <!-- 按下状态 -->
        <item android:state_pressed="true" android:drawable="@drawable/pause_on" />
        <!-- 未按下状态 -->
        <item android:state_focused="true" android:drawable="@drawable/pause_off" />
        <!-- 初始化状态 -->
        <item android:drawable="@drawable/pause_off" />    

    </selector>

pause_on.png如下图

![img](/media/pic/android/widgets/zoombutton01.png)

pasue_off如下图

![img](/media/pic/android/widgets/zoombutton02.png)
 

运行效果如下图

![img](/media/pic/android/widgets/zoombutton03.jpg)
 
