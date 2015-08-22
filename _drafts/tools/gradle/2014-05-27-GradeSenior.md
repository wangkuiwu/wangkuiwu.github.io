---
layout: post
title: "Gradle工具(三)之 Gradle高级功能"
description: ""
category: gradle
tags: [gradle]
date: 2014-05-27 11:25
---

> 本章介绍Gradle的一些有用的辅助功能。包括：导入jar包，添加Proguard代码混淆。


<a name="anchor1"></a>
# 导入jar包

导入libs/android-support-v4.jar包的方法如下

    android {

        ...

        dependencies {
            compile files('libs/android-support-v4.jar')
        }

        ...
    }


<a name="anchor2"></a>
# 添加代码混淆

    android {

        ...

        signingConfigs {
            myConfig{
                storeFile file("gradle.keystore")
                storePassword "gradle"
                keyAlias "gradle"
                keyPassword "gradle"
            }
        }
        
        buildTypes {
            release {
                // 签名信息
                signingConfig  signingConfigs.myConfig
                // 打开Proguard(默认runProguard是false)
                runProguard true
                // 设置Proguard文件。
                // 这里使用的proguard-android.txt是$sdk/tools/proguard/proguard-android.txt的副本
                proguardFile 'proguard-android.txt'
            }
        }

        ...
    }


