---
layout: post
title: "Android API指南(二)自定义控件02之 onMeasure"
description: "android training"
category: android
tags: [android]
date: 2014-06-20 10:11
---


> 本文介绍View的onMeasue方法。


<a name="anchor1"></a>
# onMeasue简介

点击查看：[onMeasure示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/self_view/03_onmeasure)

measure的作用是根据"配置文件中定义的View大小"以及"自身对View的特殊需求"来设置View的大小。onMeasure()是设置View大小的回调函数，在自定义View控件时，通常需要覆盖onMeasure来设置View的大小。


    @Override  
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {   
        setMeasuredDimension(measureWidth(widthMeasureSpec), measureHeight(heightMeasureSpec));  
    }   

    private int measureWidth(int measureSpec) {   
        int result = 0;  
        int specMode = MeasureSpec.getMode(measureSpec);  
        int specSize = MeasureSpec.getSize(measureSpec);  
  
        if (specMode == MeasureSpec.EXACTLY) {   
            result = specSize;  
        } else {   
            // Measure the text  
            result = (int) mPaint.measureText(text) + getPaddingLeft() + getPaddingRight();  
            if (specMode == MeasureSpec.AT_MOST) {   
                result = Math.min(result, specSize);
            }   
        }   
  
        return result;
    }

说明：上面是自定义的View中，覆盖的onMeasure()的相关代码。下面以该示例对onMeasure的相关知识点进行说明。

onMeasure()的参数包括了两个非常重要的信息：宽度信息(widthMeasureSpec)和高度信息(heightMeasureSpec)。这些信息都可以通过MeasureSpec解析出来。以宽度信息来说，通过MeasureSpec我们能解析出该"宽度的类型"以及"宽度的具体大小"。

## 1. 类型

类型：包括"EXACTLY"，"AT_MOST"以及"UNSPECIFIED"三种。该类型可以通过MeasureSpec.getMode()来获取。

**EXACTLY**: 父容器已经为子容器设置了尺寸。例如，在配置文件中，将View的大小设置为"固定的大小"或者"fill_parent"。  
**AT_MOST**: 子容器可以是声明大小内的任意大小。例如，在配置文件中，将View设置为"wrap_content"。  
**UNSPECIFIED**: 父容器对于子容器没有任何限制，子容器想要多大就多大。


## 2. 大小

可以通过可以通过MeasureSpec.getSize()来获取。不过需要注意的是，通过该方法获取的大小是配置文件中定义的大小。如果我们自己对该View有特殊的大小要求，则需要根据情况进行处理。



**注意**：当我们计算出View的大小之后，记得通过View的setMeasuredDimension()将计算出的大小赋值给该View。

