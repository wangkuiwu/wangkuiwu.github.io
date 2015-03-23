---
layout: post
title: "Android培训(一)开始篇04之 Activity的生命周期"
description: "android training"
category: android
tags: [android]
date: 2014-01-04 09:25
---

> 本章介绍Activity的生命周期。

> **目录**  
[1. Activity生命周期图](#anchor1)  
[2. Activity生命周期实例解说](#anchor2)  
[3. Acitivty回掉函数说明](#anchor3)  
[4. onSaveInstanceState和onRestoreInstanceState](#anchor4)  


<a name="anchor1"></a>
# 1. Activity生命周期图

Activity的生命周期图如下：

![img](/media/pic/android/training/lifecycle01.png)

说明：在Activity存在期间，只有三个可持久状态，其它的都是暂态。这三个可持久状态分别是：Resumed，Paused和Stopped。

(01) Resumed是该Activity完全可见的状态，也就是它位于前端的状态。  
(02) Paused是该Activity部分可见的状态。例如，一个Dialog对话框挡住了该Activity。  
(03) Stopped是该Activity完全不可见的状态。例如，从该Activity跳转到其他Activity，该Activity就处于Stopped状态了。  



<a name="anchor2"></a>
# 2. Activity生命周期实例解说

## 示例1--基本生命周期


    public class LifeCycle extends Activity {

        private final static String TAG = "##LifeCycle##";

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            Log.d(TAG, "onCreate");
        }   

        @Override
        public void onStart() {
            super.onStart();
            Log.d(TAG, "onStart");
        }   

        @Override
        public void onRestart() {
            super.onRestart();
            Log.d(TAG, "onRestart");
        }   

        @Override
        public void onResume() {
            super.onResume();
            Log.d(TAG, "onResume");
        }   
        @Override
        public void onPause() {
            super.onPause();
            Log.d(TAG, "onPause");
        }   

        @Override
        public void onStop() {
            super.onStop();
            Log.d(TAG, "onStop");
        }   

        @Override
        public void onDestroy() {
            super.onDestroy();
            Log.d(TAG, "onDestroy");
        } 
    }

(01). 点击进入该Activity，依次调用:  
onCreate() --> onStart() -> onResume()

(02). 进入该Activity后，按返回退出，依次调用:  
onPause() --> onStop() -> onDestroy()

(03). 进入该Activity后，点击HOME键退出(或点击"最近使用程序"进行其他程序)，依次调用:  
onPause() --> onStop() 

(04). 在(03)状态下再次进入该Activity，依次调用:  
onRestart() --> onStart() -> onResume()

点击查看：[示例一完整源码](https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/04_activity_lifecycle/01_basic_cycle/LifeCycle_01)



## 示例2--快速灭亡


    public class LifeCycle extends Activity {

        private final static String TAG = "##LifeCycle##";

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            Log.d(TAG, "onCreate");
            finish();
        }   

        @Override
        public void onStart() {
            super.onStart();
            Log.d(TAG, "onStart");
        }   

        @Override
        public void onRestart() {
            super.onRestart();
            Log.d(TAG, "onRestart");
        }   

        @Override
        public void onResume() {
            super.onResume();
            Log.d(TAG, "onResume");
        }   
        @Override
        public void onPause() {
            super.onPause();
            Log.d(TAG, "onPause");
        }   

        @Override
        public void onStop() {
            super.onStop();
            Log.d(TAG, "onStop");
        }   

        @Override
        public void onDestroy() {
            super.onDestroy();
            Log.d(TAG, "onDestroy");
        } 
    }

注意：本示例相比"上一个示例"，在onCreate()中添加了finish()。finish()的作用是主动销毁该Activity。

在安装该APK后，点击进入该Activity，依次调用:  
onCreate() --> onDestroy()

点击查看：[示例二完整源码](https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/04_activity_lifecycle/01_basic_cycle/LifeCycle_02)





<a name="anchor3"></a>
# 3. Acitivty回掉函数说明

## 3.1 onCreate()

触发条件：onCreate()是打开Activity第一个被毁掉的函数。

常见动作：通常在onCreate()中进行初始化以及显示准备。


## 3.2 onDestroy()

触发条件：当该Activity被销毁时，onDestroy()会被回调。例如，通过返回键退该Acitivty；或者，当Activity在Stopped状态下，由于资源紧张，系统主动销毁掉该Activity。

常见动作：通常在onCreate()中销毁和释放资源。

注意：onDestroy()被调用有两种情况：一种是Activity在Stopped状态被销毁；另一种是在非Stopped状态调用finish()来主动销毁该Activity。


## 3.3 onStart()

触发条件：通常在onCreate()执行完之后被调用。onStart()被调用时，Activity已经是可见的。

注意：onStart()被调用实际上有两种情况：一是onCreate()执行完后被调用；另一种是从onRestart()执行完后被调用。



## 3.4 onResume()

触发条件：在onStart()执行完之后被调用。onResume()被调用时，用户可以获取当前Activity的焦点。


常见动作：在onResume()中确保资源都正常！例如，如果在onPause()中释放了Camera，则需要在onResume()时重新初始化Camera。

注意：onResume()被调用有两种情况：一是初次进入Activity；另一种是Activity进入Stopped状态之后，再次回到前端。



## 3.5 onPause

触发条件：当Activity从Resumed状态变为Paused状态时被调用。

常见动作：在onPause()中完成以下动作。  
(01) 停止动画或其他很耗CPU的动作。  
(02) 对于需要永久保存的内容(例如，邮件的草稿)，在onPause()中进行保存。  
(03) 释放使用电池的资源，例如GPS、Camera的句柄等。  



## 3.6 onStop

触发条件：当Activity从Paused状态变为Stopped状态时被调用。  
常见的情况有：  
(01) 用户通过"最近常用程序"切换到其他Activity。  
(02) 在当前Activity启动并进入了另一个Activity。   
(03) 当Activity在前端时，来电话并弹出了拨号界面。   
(04) 当Activity在前端时，按HOME键返回主界面。   

常见动作：调用了onStop()意味着该Activity完全不可见，通常在onStop()中释放一切可能造成内存泄漏的资源。  




## 3.7 onRestart()

触发条件：当Activity在Stopped状态再次被显示到前端时被调用。

常见动作：Google官网并没有给出onRestart()的指导动作。因为它接着会调用onStart()，这就意味着，即使有动作，也可以将它们放到onStart()中去完成。






<a name="anchor4"></a>
# 4. onSaveInstanceState和onRestoreInstanceState

onSaveInstanceState()是用于保存数据，onRestoreInstanceState()用于还原数据。

## 它在什么情况会被触发呢？
当该Activity会完全隐藏，但是又没有被销毁时，会触发onSaveInstanceState()。也就是说以下几种情况会触发onSaveInstanceState()：
(01) 用户通过"最近常用程序"切换到其他Activity。  
(02) 在当前Activity启动并进入了另一个Activity。   
(03) 当Activity在前端时，来电话并弹出了拨号界面。   
(04) 当Activity在前端时，按HOME键返回主界面。   

如果"用户在Activity界面按返回退出Activity" 或者 "通过finish()销毁掉该Activity"，并不会触发onSaveInstanceState()。


## 它的作用是什么

通常我们在onSaveInstanceState()保存Activity的状态。  

举例来说，假如你在打游戏(处于一个Activity界面)，此时正好电话打进来了；Android会触发onSaveInstanceState()。此时，你可以将你想要的数据保存起来。电话结束之后，如果由于系统资源不足等原因导致该Activity被销毁了，那么你再次进入Activity，onCreate(savedInstanceState)中的savedInstanceState就不为空，它就是在onSaveInstanceState()中所保存的数据。此时，你可以根据保存的数据来还原之前的游戏进度！

除了在onCreate()中，你也可以选择在onRestoreInstanceState()中来还原之前的游戏进度。二者选其一即可！  
需要注意的是，在onCreate()中需要对savedInstanceState是否为null进行判断，而onRestoreInstanceState()中的savedInstanceState肯定不是null！


此外，如果电话结束之后，不是由于系统不足导致Activity被销毁，而是你主动销毁，比如通过"最近使用程序"窗口将该应用关闭。那么，之前通过onSaveInstanceState()保存的数据就被销毁了；即，当你再次进入Activity时，onCreate(savedInstanceState)中的savedInstanceState为null。


总的来说，  
(01) onSaveInstanceState()是用于保存数据，onRestoreInstanceState()用于还原数据。它们可以配对使用，但不是绝对的；onRestoreInstanceState()的工作可以在onCreate()中完成。  
(02) 只有当Activity被完全隐藏，并且Activity没有被销毁的情况下，onSaveInstanceState()才会被调用。  
(03) 只有当onSaveInstanceState()被调用，并且该Activity由于资源不足等原因被系统销毁的情况下；重新进入Activity时，onRestoreInstanceState()才会被调用！  



