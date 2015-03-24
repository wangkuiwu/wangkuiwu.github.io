---
layout: post
title: "Android控件篇10之 ZoomControls"
description: "android training"
category: android
tags: [android]
date: 2014-02-10 09:11
---





<a name="anchor1"></a>
# 1. ZoomControls简介

ZoomButton是一个放大缩小按钮。

点击它的放大按钮，它能不断的上报放大事件；点击它的缩小按钮，它能不断的上报缩小事件。上报的时间间隔可以控制，而且ZoomButton可以隐藏。



<a name="anchor2"></a>
# 2. ZoomControls示例

写一个activity，包含一个ZoomControls。点击ZoomControls，能够缩放文字。

**应用层代码**

    package com.skywang.control;

    import android.os.Bundle;
    import android.app.Activity;
    import android.view.Menu;
    import android.view.View;
    import android.view.View.OnClickListener;
    import android.widget.TextView;
    import android.widget.ZoomControls;

    public class ZoomControlsTest extends Activity {

        private static int pauseSize= 14;
        private ZoomControls mZC;
        private TextView mTvShow;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.zoom_controls_test);
            

            mTvShow = (TextView)findViewById(R.id.tv_show);
            mZC = (ZoomControls)findViewById(R.id.zc_one);
            //开启ZoomControls放大功能
            mZC.setIsZoomInEnabled(true);
            // 开启ZoomControls缩小功能
            mZC.setIsZoomOutEnabled(true);
            //图片放大
            mZC.setOnZoomInClickListener(new OnClickListener() {
                public void onClick(View v) {
                    pauseSize += 2;
                    mTvShow.setTextSize(pauseSize);
                }
            });
            //图片缩小
            mZC.setOnZoomOutClickListener(new OnClickListener() {
                public void onClick(View v) {
                    pauseSize -= 2;
                    mTvShow.setTextSize(pauseSize);
                }
            });
        }
    }

**layout代码**

    <ZoomControls
        android:id="@+id/zc_one"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>


运行效果：如下图

![img](/media/pic/android/widgets/zoomcontrols01.jpg)

