---
layout: post
title: "Android培训(二)共享篇01之 共享简单数据"
description: "android training"
category: android
tags: [android]
date: 2014-01-11 09:25
---

> 本章介绍如何通过Intent来共享简单数据

> **目录**  
[1. 共享数据简介](#anchor1)  
[2. ActionBar共享支持](#anchor2)  


<a name="anchor1"></a>
# 1. 共享数据简介

通过Intent共享数据，主要涉及到的几个方面：发送数据、接收数据、发送/返回结果。这些内容在介绍[App的交互类Intent](/2014/05/31/Intent)时，已经介绍过了，这里就不再重复说明！



<a name="anchor2"></a>
# 2. ActionBar共享支持

如果你想在ActionBar中添加"分享"功能。有个很简单的方法：Android默认提供了对该功能的支持。

步骤一：在menu的配置菜单中添加android:actionProviderClass="android.widget.ShareActionProvider"的配置项目。如下示例：

    <menu xmlns:android="http://schemas.android.com/apk/res/android">
        <item
                android:id="@+id/menu_item_share"
                android:showAsAction="ifRoom"
                android:title="Share"
                android:actionProviderClass=
                    "android.widget.ShareActionProvider" />
        ...
    </menu>


步骤二：在Activity中通过MenuItem的getActionProvider()获取对应的ShareActionProvider对象。获取ShareActionProvider之后，可以通过setShareIntent(intent)来设置点击该按钮的触发事件。


    private ShareActionProvider mShareActionProvider;
    ...

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate menu resource file.
        getMenuInflater().inflate(R.menu.share_menu, menu);

        // Locate MenuItem with ShareActionProvider
        MenuItem item = menu.findItem(R.id.menu_item_share);

        // Fetch and store ShareActionProvider
        mShareActionProvider = (ShareActionProvider) item.getActionProvider();

        // Return true to display menu
        return true;
    }

    // Call to update the share intent
    private void setShareIntent(Intent shareIntent) {
        if (mShareActionProvider != null) {
            mShareActionProvider.setShareIntent(shareIntent);
        }

点击查看：[ActionBar分享示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/training/02_sharing_data/01_share_simple_data/03_add_easy_share_action/SendText)


