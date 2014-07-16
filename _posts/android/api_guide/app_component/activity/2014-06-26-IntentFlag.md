---
layout: post
title: "Android 之Activity启动模式(二)之 Intent的Flag属性"
description: "android training"
category: android
tags: [android]
date: 2014-06-26 10:12
---

> 前面介绍了通过launchMode设置Activity的启动模式。本章接着介绍Activity的启动模式相关内容，讲解的内容是Intent与启动模式相关的Flag，以及android:taskAffinity的属性。

> **目录**  
> **1**. [Intent与启动模式相关的Flag简介](#anchor1)  
> **2**. [1. FLAG_ACTIVITY_NEW_TASK标签测试](#anchor2)  
> **3**. [2. FLAG_ACTIVITY_CLEAR_TOP标签测试](#anchor3)  
> **4**. [3. FLAG_ACTIVITY_CLEAR_TASK标签测试](#anchor4)  
> **5**. [4. FLAG_ACTIVITY_SINGLE_TOP标签测试](#anchor5)  


<a name="anchor1"></a>
# Intent与启动模式相关的Flag简介

这里仅仅对几个常用的与启动模式相关的Flag进行介绍。


1. **FLAG_ACTIVITY_NEW_TASK**  
  在google的官方文档中介绍，它与launchMode="singleTask"具有相同的行为。实际上，并不是完全相同！  
  很少单独使用FLAG_ACTIVITY_NEW_TASK，通常与FLAG_ACTIVITY_CLEAR_TASK或FLAG_ACTIVITY_CLEAR_TOP联合使用。因为单独使用该属性会导致奇怪的现象，通常达不到我们想要的效果！尽管如何，后面还是会通过"FLAG_ACTIVITY_NEW_TASK示例一"和"FLAG_ACTIVITY_NEW_TASK示例二"会向你展示单独使用它的效果。


2. **FLAG_ACTIVITY_SINGLE_TOP**  
  在google的官方文档中介绍，它与launchMode="singleTop"具有相同的行为。实际上，的确如此！单独的使用FLAG_ACTIVITY_SINGLE_TOP，就能达到和launchMode="singleTop"一样的效果。

3. **FLAG_ACTIVITY_CLEAR_TOP**  
  顾名思义，FLAG_ACTIVITY_CLEAR_TOP的作用清除"包含Activity的task"中位于该Activity实例之上的其他Activity实例。FLAG_ACTIVITY_CLEAR_TOP和FLAG_ACTIVITY_NEW_TASK两者同时使用，就能达到和launchMode="singleTask"一样的效果！

4. **FLAG_ACTIVITY_CLEAR_TASK**  
  FLAG_ACTIVITY_CLEAR_TASK的作用包含Activity的task。使用FLAG_ACTIVITY_CLEAR_TASK时，通常会包含FLAG_ACTIVITY_NEW_TASK。这样做的目的是启动Activity时，清除之前已经存在的Activity实例所在的task；这自然也就清除了之前存在的Activity实例！

注意：**当同时使用launchMode和上面的FLAG_ACTIVITY_NEW_TASK等标签时，以FLAG_ACTIVITY_NEW_TASK为标准。也就是说，代码的优先级比manifest中配置文件的优先级更高**！ 

下面，通过几个实例加深对这几个标记的理解。


<a name="anchor2"></a>
# 1. FLAG_ACTIVITY_NEW_TASK标签测试

## 1.1 FLAG_ACTIVITY_NEW_TASK示例一

点击查看：[FLAG_ACTIVITY_NEW_TASK示例一的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/02_new_task/01_same_taskAffinity)

在该实例中，有两个Activity：ActivityTest和SecondActivity。manifest定义如下：


    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity" />
    </application>

说明：通过manifest可以看出，ActivityTest和SecondActivity在同一个APK中。这也就意味着它们的android:taskAffinity是一样的！


**ActivityTest的源码**

    public class ActivityTest extends Activity {
        private static final String TAG="##ActivityTest##";

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            Log.d(TAG, "onCreate: "+this.toString()+", taskId="+this.getTaskId());
            TextView tv = (TextView) findViewById(R.id.tv);
            tv.setText(this.toString()+", taskId="+this.getTaskId());
        }   

        public void onJump(View view) {
            Intent intent = new Intent(this, SecondActivity.class);
            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
            startActivity(intent);
        }   

        @Override
        protected void onNewIntent(Intent intent) {
            Log.d(TAG, "onNewIntent: intent="+intent+", activity="+this+", taskId="+this.getTaskId());
        }   
    }

说明：onJump()是ActivityTest中一个按钮的回调函数，点击该按钮会跳转到SecondActivity。**注意，跳转的Intent添加了FLAG_ACTIVITY_NEW_TASK标志**。


**SecondActivity的源码**

    public class SecondActivity extends Activity {

        private static final String TAG="##SecondActivity##";
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.second);

            Log.d(TAG, "onCreate: "+this.toString()+", taskId="+this.getTaskId());
            TextView tv = (TextView) findViewById(R.id.tv2);
            tv.setText(this.toString()+", taskId="+this.getTaskId());
        }   

        public void onBack(View view) {
            Intent intent = new Intent(this, ActivityTest.class);
            startActivity(intent);
        }   

        @Override
        protected void onNewIntent(Intent intent) {
            Log.d(TAG, "onNewIntent: intent="+intent+", activity="+this+", taskId="+this.getTaskId());
        }   
    }

说明：onBack()是SecondActivity中一个按钮的回调函数，点击该按钮会跳转回ActivityTest。

**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity  
**测试结果**：(01) ActivityTest和SecondActivity在同一个task中。 (02) 两个SecondActivity不同的实例！  
**结果分析**：如果说FLAG_ACTIVITY_NEW_TASK的作用和singleTask具有相同的效果。那么这个示例很明显的否则了这个结论！事实上，在相互跳转的两个Activity的android:taskAffinity相同的情况下，单独使用FLAG_ACTIVITY_NEW_TASK不会产生任何效果！

那如果两个Activity的android:taskAffinity不相同呢？此时会导致什么效果呢？下面，我们通过示例来看看效果。



## 1.2 FLAG_ACTIVITY_NEW_TASK示例二

点击查看：[FLAG_ACTIVITY_NEW_TASK示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/02_new_task/02_diff_taskAffinity)

我们修改"FLAG_ACTIVITY_NEW_TASK示例一"中manifest，将ActivityTest和SecondActivity的android:taskAffinity改为不同；其余的保持不变！修改后的manifest如下：

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:taskAffinity="com.skw.activitytest01"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity"
                  android:taskAffinity="com.skw.activitytest02"
            />
    </application>

**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity  
**测试结果**：(01) ActivityTest和SecondActivity在不同task中！ (02) 当第二次进入到ActivityTest中，再企图从ActivityTest中进入到SecondActivity时，没有产生任何效果，仍然停留在ActivityTest中！即第二次ActivityTest --> SecondActivity压根就没发生！    
**结果分析**：当相互跳转的两个Activity的android:taskAffinity不同时，添加FLAG_ACTIVITY_NEW_TASK确实产生了一些效果：第一次启动Activity时，会新建一个task，并将Activity添加到该task中。这与singleTask产生的效果是一样的！但是，当企图再次从ActivityTest进入到SecondActivity时，却什么也没有发生！  
为什么呢？是因为此时SecondActivity实例已经存在，但是它所在的task的栈顶是ActivityTest；而单独的添加FLAG_ACTIVITY_NEW_TASK又不会"删除task中位于SecondActivity之上的Activity实例"，所以就没有发生跳转！  

好的，那下面，我们添加FLAG_ACTIVITY_CLEAR_TOP之后，再来看看效果。



<a name="anchor3"></a>
# 2. FLAG_ACTIVITY_CLEAR_TOP标签测试

## 2.1 FLAG_ACTIVITY_CLEAR_TOP示例一

点击查看：[FLAG_ACTIVITY_CLEAR_TOP示例一的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/03_clear_top/01_same_taskAffinity)。

我们修改"FLAG_ACTIVITY_NEW_TASK示例一"中onJump()函数，修改后的代码如下：


    public void onJump(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_CLEAR_TOP);
        startActivity(intent);
    }   

**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity  
**测试结果**：(01) ActivityTest和SecondActivity在同一个task中！ (02) 两个SecondActivity是不同的实例。  
**结果分析**：这与没有添加FLAG_ACTIVITY_CLEAR_TOP时效果一样！这说明，当相互跳转的两个Activity的android:taskAffinity一样时，不会产生任何效果！  

接下来，看看不同android:taskAffinity的情况。


## 2.2 FLAG_ACTIVITY_CLEAR_TOP示例二

点击查看：[FLAG_ACTIVITY_CLEAR_TOP示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/03_clear_top/02_diff_taskAffinity)。

我们修改"FLAG_ACTIVITY_NEW_TASK示例一"中onJump()函数，修改后的代码如下：

    public void onJump(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_CLEAR_TOP);
        startActivity(intent);
    }   

**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity  
**测试结果**：(01) ActivityTest和SecondActivity在不同task中！ (02) 两个SecondActivity是同一个实例。  
**结果分析**：此时的表现和SecondActivity是singleTask一样！ 这说明，在相互跳转的Activity的android:taskAffinity不同时，同时使用FLAG_ACTIVITY_NEW_TASK和FLAG_ACTIVITY_CLEAR_TOP，才具有和singleTask一样的效果！

总的来说：FLAG_ACTIVITY_NEW_TASK和FLAG_ACTIVITY_CLEAR_TOP的使用和android:taskAffinity相关。在同时使用FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TOP的情况下，以A启动B来说   
(01) 当A和B的taskAffinity相同时：添加FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TOP没有任何作用。和没有添加时的效果一样！  
(02) 当A和B的taskAffinity不同时：添加FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TOP后，表现的和B是singleTask一样！




<a name="anchor4"></a>
# 3. FLAG_ACTIVITY_CLEAR_TASK标签测试

## 3.1 FLAG_ACTIVITY_CLEAR_TASK示例一

点击查看：[FLAG_ACTIVITY_CLEAR_TASK示例一的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/04_clear_task/01_same_taskAffinity)


我们修改"FLAG_ACTIVITY_NEW_TASK示例一"中onJump()函数，修改后的代码如下：

    public void onJump(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        startActivity(intent);
    }   

**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity  
**测试结果**：(01) ActivityTest和SecondActivity在同一个task中！ (02) 两个SecondActivity是不同的实例。  
**结果分析**：这与没有添加FLAG_ACTIVITY_CLEAR_TASK时效果一样！这说明，当相互跳转的两个Activity的android:taskAffinity一样时，不会产生任何效果！  


接下来，看看不同android:taskAffinity的情况。


## 3.2 FLAG_ACTIVITY_CLEAR_TASK示例二

点击查看：[FLAG_ACTIVITY_CLEAR_TASK示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/04_clear_task/02_diff_taskAffinity)

我们修改"FLAG_ACTIVITY_NEW_TASK示例一"中onJump()函数，修改后的代码如下：

    public void onJump(View view) {
        Intent intent = new Intent(this, SecondActivity.class);
        intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                | Intent.FLAG_ACTIVITY_CLEAR_TASK);
        startActivity(intent);
    }   

**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity --> 返回键 --> 返回键  
**测试结果**：(01) ActivityTest和SecondActivity在不同的task中！ (02) 两个SecondActivity是不同的实例。 (03) 第一次返回键，返回到第一个ActivityTest中。 (04) 第二次返回键，返回到进入第一个ActivityTest之前的画面。   
**结果分析**：当同时使用FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TASK时，每次启动Activity时，若该Activity的实例已经存在于某个task中，则清除该task中的全部内容；然后重新创建task并将Activity添加到新建的task中；否则，直接启动新的task并将该Activity添加到新建的task中。



总的来说：FLAG_ACTIVITY_NEW_TASK和FLAG_ACTIVITY_CLEAR_TASK的使用和android:taskAffinity相关。在同时使用FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TASK的情况下，以A启动B来说   
(01) 当A和B的taskAffinity相同时：添加FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TASK没有任何作用。和没有添加时的效果一样！  
(02) 当A和B的taskAffinity不同时：添加FLAG_ACTIVITY_NEW_TASK|FLAG_ACTIVITY_CLEAR_TASK后，启动B时，若该B已经存在于某个task中，则清除该task中的全部内容；然后重新创建task并将B添加到新建的task中；否则，直接启动新的task并将B添加到新建的task中。



<a name="anchor5"></a>
# 4. FLAG_ACTIVITY_SINGLE_TOP标签测试

FLAG_ACTIVITY_SINGLE_TOP的特性和launchMode="singleTop"一样！这里就不做过多的说明了。


点击查看：[FLAG_ACTIVITY_SINGLE_TOP示例一的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/01_single_top/01_single)。该示例中，只有一个Activity示例，点击该Activity会跳转到它自身。

点击查看：[FLAG_ACTIVITY_SINGLE_TOP示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/intent_lauchmode/01_single_top/02_same_taskAffinity)。该示例中，有两个Activity示例，两个Activity之间可以相互跳转。


