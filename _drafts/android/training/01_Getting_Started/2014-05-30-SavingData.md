---
layout: post
title: "Android培训(一)开始篇06之 保存数据"
description: "android training"
category: android
tags: [android]
date: 2014-05-30 19:25
---

> 本章Fragment

> **目录**  
> **1**. [SharedPreferences保存数据](#anchor1)  
> **2**. [File保存数据](#anchor2)  
> **3**. [SQLite保存数据](#anchor3)  


<a name="anchor1"></a>
# SharedPreferences保存数据

## 1. 基本用法

### 获取SharedPreferences对象

在Activity中有两种获取SharedPreferences的方法：   
(01) getSharedPreferences(name, mode) -- 该方法能够指定SharedPreferences的名字。  
(02) getPreferences(mode) -- 该方法会采用该Activity的名称作为名字。  
注意：不同Activity调用getPreferences()获取到的SharedPreferences并不相同！

        // 获取SharedPreferences方法一：指定名称
        mPref = getSharedPreferences(SP_FILE_NAME, MODE_PRIVATE); 
        // 获取SharedPreferences方法二：apk默认
        mPref = getPreferences(MODE_PRIVATE);


### 写数据

首先获取SharedPreferences.Editor对象，通过该对象才能写入数据。

    mEditor = mPref.edit();

接着，就可以通过mEditor来将数据保存到SharedPreferences中。SharedPreferences中的数据是以key-value键值对的形式存在的。另外，添加了数据之后，记得调用commit()接口提交数据。

    private void writePref() {
        mEditor.putString(SP_NAME, "jim");      // 名字
        mEditor.putInt(SP_AGE, 25);             // 年龄
        mEditor.putBoolean(SP_SINGLE, false);   // 婚否

        // 提交数据
        mEditor.commit();
    }   

### 读数据

直接调用SharedPreferences的相应接口就能读取数据。

    private void readPref() {
        String name = mPref.getString(SP_NAME, null); 
        int age = mPref.getInt(SP_AGE, 1); 
        boolean single = mPref.getBoolean(SP_SINGLE, false); 
    }   

点击查看：[SharedPreferences工程的完整源码][https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/06_saving_data/01_shared_preferences/01_basic]





<a name="anchor2"></a>
# File保存数据

将数据存储成文件，并保存到设备上。可以选择Exernal(外部) 和 Internal(内部)存储设别。

(01) Exernal





<a name="anchor3"></a>
# SharedPreferences保存数据

