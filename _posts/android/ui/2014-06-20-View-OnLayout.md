---
layout: post
title: "Android API指南(二)自定义控件03之 onLayout"
description: "android training"
category: android
tags: [android]
date: 2014-06-20 11:11
---


> 本文介绍View的onLayout方法。onLayout通常在View给其孩子设置尺寸和位置时被调用。


<a name="anchor1"></a>
# onLayout简介

点击查看：[onLayout示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/self_view/05_viewgroup/ViewTest)

在view给其孩子设置尺寸和位置时被调用。在我们自定义ViewGroup时，通常需要覆盖该方法。示例如下：


    public class MyGroupView extends ViewGroup {

        public MyGroupView(Context context) {
            super(context);
        }

        public MyGroupView(Context context, AttributeSet attrs) {
            super(context, attrs);
        }

        public MyGroupView(Context context, AttributeSet attrs, int defStyle) {
            super(context, attrs, defStyle);
        }

        ...

        @Override
        protected void onLayout(boolean changed, int l, int t, int r, int b) {
            // 记录总高度
            int mTotalHeight = 0;
            // 遍历所有子视图
            int childCount = getChildCount();
            for (int i = 0; i < childCount; i++) {
                View childView = getChildAt(i);

                // 获取在onMeasure中计算的视图尺寸
                int measureHeight = childView.getMeasuredHeight();
                int measuredWidth = childView.getMeasuredWidth();

                childView.layout(l, mTotalHeight, measuredWidth, mTotalHeight
                        + measureHeight);

                mTotalHeight += measureHeight;
            }
        }
    }


自定义了ViewGroup之后，就可以使用该ViewGroup了。示意如下：

    <RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="#00f0f0"
        tools:context=".MainActivity" >

        <com.skw.viewtest.MyGroupView
            android:id="@+id/myViewGroup"
            android:layout_width="480dp"
            android:layout_height="300dp"
            android:background="#0f0f0f" >

            <TextView
                android:layout_width="fill_parent"
                android:layout_height="60dp"
                android:background="#000000"
                android:gravity="center"
                android:text="First TextView" />

            <TextView
                android:layout_width="200dp"
                android:layout_height="100dp"
                android:background="#ffffff"
                android:gravity="center"
                android:text="Second TextView" />
        </com.skw.viewtest.MyGroupView>

    </RelativeLayout>


