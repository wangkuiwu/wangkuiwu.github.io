---
layout: post
title: "Gradle工具(二)之 使用Gradle管理Android工程"
description: ""
category: gradle
tags: [gradle]
date: 2014-05-27 10:25
---

> 本文介绍利用Grade管理Android工程的方法。


<a name="anchor1"></a>
# 搭建Java开发环境


**第一步**：创建Android工程

    android create project \
    --target 1 \
    --name MyAndroidApp \
    --path ./MyAndroidAppProject \
    --activity MyAndroidAppActivity \
    --package com.example.myandroid



**第二步**：配置Gradle文件

在MyAndroidApp工程下新建build.gradle文件，并输入以下内容：



    buildscript {
        repositories {
            mavenCentral()
        }   
        dependencies {
            classpath 'com.android.tools.build:gradle:0.8.+'
        }   
    }

    apply plugin: 'android'

    android {
        compileSdkVersion 19
        buildToolsVersion "19.0.3"

        sourceSets {
            main {
                manifest {
                    srcFile 'AndroidManifest.xml'
                }   
                java {
                    srcDir 'src'
                }   
                res {
                    srcDir 'res'
                }   
                assets {
                    srcDir 'assets'
                }   
                resources {
                    srcDir 'src'
                }   
                aidl {
                    srcDir 'src'
                }   
            }   
        }   
    }



**第三步**：编译

通过"gradle build"或"gradle -q"来编译工程，编译完成之后会在工程下新生成一个build文件夹。



**第四步**：安装

编译完毕之后，可以通过"gradle installDebug"来安装。


**gradle tasks**: 查看在gradle指令。
**gradle clean**: 清空工程。
**gradle build**: 编译工程。
**gradle -q**: 编译工程，只打印出错信息。
**gradle installDebug**: 安装debug版本的apk到设备上。


