---
layout: post
title: "Android 之ContentProvider(三)之 Permission权限设置"
description: "android training"
category: android
tags: [android]
date: 2014-07-06 12:11
---

> 前面自定义Permission权限。自定义权限除了用在ContentProvider中之外，也可以用在Activity与Service中。

> **目录**  
> **1**. [Permission介绍](#anchor1)  
> **2**. [自定义Permission示例](#anchor2)  


<a name="anchor1"></a>
# Permission介绍

    <permission
        android:name="com.skw.permission.myprovider"
        android:protectionLevel="normal"
        android:label="@string/permission_label"
        android:description="@string/permission_description"
        />


说明：permission常用的几个属性如下：  
(01) **android:name**: 必需的。权限的名称，通常应遵循android 命名方案(*.permission.*)。  
(02) **android:protectionLevel**: 必需的。权限的安全级别，共包括"normal, dangerous, signature, signatureOrSystem"四种。  
> normal 表示权限是低风险的，不会对系统、用户或其他应用程序造成危害；  
> dangerous 表示权限是高风险的，系统将可能要求用户输入相关信息，才会授予此权限；  
> signature 表示只有当应用程序所用数字签名与声明引权限的应用程序所用数字签名相同时，才能将权限授给它；  
> signatureOrSystem 表示将权限授给具有相同数字签名的应用程序或android 包类。

(03) **android:permissionGroup**: 非必需的。可以将权限放在一个组中，但对于自定义权限，应该避免设置此属性。  
(04) **android:label**: 非必需的。标签。  
(05) **android:description**: 非必需的。描述。  
(06) **android:icon**非必需的。图标。  



<a name="anchor2"></a>
# 自定义Permission示例

点击查看：[自定义Permission完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/contentprovider/02_permission/MyProvider)

## 1. 设置权限

    <application 
        android:exported="true"
        android:label="@string/app_name" 
        android:icon="@drawable/ic_launcher">
        <provider 
            android:name="MyProvider"
            android:authorities="com.skw.myprovider"
            android:permission="com.skw.permission.myprovider">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </provider>
    </application>

    <permission
        android:name="com.skw.permission.myprovider"
        android:protectionLevel="normal"
        android:label="@string/permission_label"
        android:description="@string/permission_description"
        />

说明：上面是自定义权限的manifest文件内容。  
(01) 首先，在需要定义权限的ContentProvider中声明权限android:permission="com.skw.permission.myprovider"。  
(02) 接着，再定义permission。permission对应的name与声明的权限对应。 

## 2. 获取权限

在需要使用ContentProvider的APK的manifest中需要声明使用该权限。声明方法如下：

    <uses-permission android:name="com.skw.permission.myprovider" />


