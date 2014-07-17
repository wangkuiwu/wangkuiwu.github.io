---
layout: post
title: "Android 之ContentProvider(一)之 Uri介绍"
description: "android training"
category: android
tags: [android]
date: 2014-07-06 09:11
---

> 本章介绍Uri及其相关内容。

> **目录**  
> **1**. [URI简介](#anchor1)  
> **2**. [Content URIs介绍](#anchor2)  


<a name="anchor1"></a>
# URI简介

URI(Universal Resource Identifier)，又被称为"通用资源标志符"。

URI由许多部分所组成，示例及解说如下：

<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/app_components/contentprovider/pic/uri01.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/app_components/contentprovider/pic/uri01.jpg" alt="" /></a>


 
<a name="anchor2"></a>
# Content URIs介绍

Android遵循URI的标准，定义了一套专用的Uri(即，Content URIs)。并且，Android提供了ContentUris、UriMatcher等类用于操作Content URIs。

 
## 1. Content URIs语法

Content URIs的语法如下：

**content://authority/path/id**


Content URIs的示例及说明如下：
 
<a href="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/app_components/contentprovider/pic/uri02.jpg"><img src="https://raw.githubusercontent.com/wangkuiwu/android_applets/master/api_guide/app_components/contentprovider/pic/uri02.jpg" alt="" /></a>


说明：  
**content**: Content URIs前缀，它对应与标准URI的scheme。它的值为ContentResolver.SCHEME_CONTENT(即，content://)。  
**authority**: 一个唯一的标识符，Google建议使用类的全名来作为authority。外部调用者可以根据这个标识来找到它。  
**path**: 它可以用来表示我们要操作的数据，外部调用者根据这个路径信息来判断要返回什么类型的数据。这个后缀路径可以自由定义。  
**id**: 唯一的数字标识符。它表示要具体操作的数据类型中的具体某一项。


## 2. Content URIs相关类介绍

### 2.1 ContentUris

ContentUris中包含了三个静态函数:  
>  long parseId(Uri uri): 解析Uri中的末尾id。成功返回id，失败则返回-1。  
>  Uri withAppendedId(Uri uri, long id): 将id追加到uri中，并返回追加id后的uri。  
>  Uri.Builder appendId(Uri.Builder builder, long id): 将id追加到builder中，并返回追加id后的builder。  


### 2.2 UriMatcher

UriMatcher用于匹配Uri。它的用法如下：  
(01) 创建UriMatcher对象。  
(02) 把你需要匹配Uri路径通过addURI()注册到UriMatcher对象上。  
(03) 注册成功后，ContentProvider就可以通过UriMatcher监听你注册的Uri。当有匹配的Uri动作(如插入)时，再就可以通过UriMatcher的match()函数来获取Uri的一个标识，该标识是在addURI()时传入的。这个标识的作用是方便在switch语句中对不同的Uri进行处理。  

 
UriMatcher的主要API说明：  
> void addURI(String authority, String path, int code): 将"authority"+"path"注册的Uri注册到UriMatcher中，code是该Uri对应的标识。  
> int match(Uri uri): 匹配Uri，并返回Uri对应的标识。这里返回的标识与addURI中的标识对应。

