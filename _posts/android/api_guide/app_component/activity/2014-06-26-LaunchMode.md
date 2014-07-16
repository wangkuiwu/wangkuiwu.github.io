---
layout: post
title: "Android 之Activity启动模式(一)之 lauchMode"
description: "android training"
category: android
tags: [android]
date: 2014-06-26 09:10
---

> 本章介绍Activity的四种launchMode。

> **目录**  
> **1**. [launchMode简介](#anchor1)  
> **2**. [1. standard模式](#anchor2)  
> **3**. [2. singleTop模式](#anchor3)  
> **4**. [3. singleTask模式](#anchor4)  
> **5**. [4. singleInstance模式](#anchor5)  
> **6**. [模式总结](#anchor6)  


<a name="anchor1"></a>
# launchMode简介

在讲解launchMode之前，需要先了解两个概念：task和taskAffinity。

task是一个"First In Last Out"的栈，task可以有一个或多个Activity。我们可以将task看作是管理Activity的单元。某一时刻，系统可以有多个task；每个task可以有一个或多个Activity。同一个Activity可能只允许存在一个实例，也可能可以有多个实例，而且这些实例既可以位于同一个task，也可以位于不同的task。Activity究竟是怎么处理它的实例，以及它在task中的分布情况；这些都可以通过launchMode进行设置。


android:taskAffinity是Activity的一个属性。例如,android:taskAffinity="string"。它的作用是描述了不同Activity之间的亲密关系。拥有相同的taskAffinity的Activity是亲密的，它们之间在相互跳转时，会位于同一个task中，而不会新建一个task！  
如果在manifest中没有对Activity的android:taskAffinity进行配置，则每个Activity都采用和Application相同的taskAffinity；这也就意味着，同一个Application中的所有Activity的taskAffinity在默认情况下是相同的！

下面开始介绍四种launchMode模式，在通过示例介绍之后，再来对这四种launchMode进行总结。


<a name="anchor2"></a>
# 1. standard模式 

standard模式是默认模式。在该模式下，Activity可以拥有多个实例，并且这些实例既可以位于同一个task，也可以位于不同的task。

下面通过示例来对standard进行验证。在该示例中，ActivityTest是standard模式的，而且点击ActivityTest中的按钮能跳转到它自身。

点击查看：[standard模式的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/01_standard)

manifest源码

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

ActivityTest的代码

    public class ActivityTest extends Activity {
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            TextView tv = (TextView) findViewById(R.id.tv);
            tv.setText(this.toString()+", taskId="+this.getTaskId());
        }   

        public void onJump(View view) {
            Intent intent = new Intent(this, ActivityTest.class);
            startActivity(intent);
        }   

        @Override
        protected void onNewIntent(Intent intent) {
            Log.d(TAG, "onNewIntent: intent="+intent);
        }   
    }

说明：上面的ActivityTest就是android:launchMode="standard"模式的。onJump()是按钮的回调函数，点击该按钮，会重新创建一个ActivityTest实例。

**测试内容**：ActivityTest --> ActivityTest --> ActivityTest。 (注：ActivityTest --> ActivityTest表示从ActivityTest跳转到ActivityTest)  
**测试结果**：每一个ActivityTest实例都是不同的，而且这三个ActivityTest实例都位于同一个task中。  
**结果分析**：这与我们前面介绍的standard模式的特性是相符的。




<a name="anchor3"></a>
# 2. singleTop模式

singleTop模式下，在同一个task中，如果存在该Activity的实例，并且该Activity实例位于栈顶(即，该Activity位于前端)，则调用startActivity()时，不再创建该Activity的示例；而仅仅只是调用Activity的onNewIntent()。否则的话，则新建该Activity的实例，并将其置于栈顶。


## 2.1 singleTop示例一

点击查看：[singleTop示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/02_single_top/SingleActvity)

我们将"standard示例"中ActivityTest的launchMode修改为singleTop，其他的保持不变。修改后的manifest如下：

        <activity android:name="ActivityTest"
                  android:launchMode="singleTop"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

说明：上面的ActivityTest就是android:launchMode="singleTop"模式的。并不会创建新的ActivityTest实例；但是会调用onNewIntent()。
        </activity>

**测试内容**：ActivityTest --> ActivityTest --> ActivityTest  
**测试结果**：每一个ActivityTest实例都是相同的！当从一个ActivityTest跳转到它自身时，没有创建新的ActivityTest实例，但是会调用onNewIntent()。  
**结果分析**：这与我们前面介绍的singleTop模式的特性是相符的。



如果是singleTop模式的Activity不在栈顶，那会如何呢？我们通过下面的示例来进行分析。


## 2.2 singleTop示例二


在该示例中，有两个Activity：ActivityTest和SecondActivity。其中ActivityTest是singleTop类型的，而SecondActivity则是standard类型的。这两个Activity之间能相互跳转。

点击查看：[singleTop示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/02_single_top/TwoActivity)

manifest的源码

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:launchMode="singleTop"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity"
                  android:launchMode="standard"/>
    </application>


ActivityTest的代码

    public class ActivityTest extends Activity {
        private static final String TAG="##ActivityTest##";

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            Log.d(TAG, "onCreate: "+this.toString());
            TextView tv = (TextView) findViewById(R.id.tv);
            tv.setText(this.toString());
        }   

        public void onJump(View view) {
            Log.d(TAG, "onJump: "+this.toString());
            Intent intent = new Intent(this, SecondActivity.class);
            startActivity(intent);
        }   

        @Override
        protected void onNewIntent(Intent intent) {
            Log.d(TAG, "onNewIntent: intent="+intent+", activity="+this);
        }   

    }

说明：上面的ActivityTest就是android:launchMode="singleTop"模式的。onJump()是按钮的回调函数，点击该按钮，会跳转到SecondActivity中。

    public class SecondActivity extends Activity {

        private static final String TAG="##SecondActivity##";
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.second);

            Log.d(TAG, "onCreate: "+this.toString());
            TextView tv = (TextView) findViewById(R.id.tv2);
            tv.setText(this.toString());
        }   

        public void onBack(View view) {
            Log.d(TAG, "onBack: "+this.toString());
            Intent intent = new Intent(this, ActivityTest.class);
            startActivity(intent);
        }   

        @Override
        protected void onNewIntent(Intent intent) {
            Log.d(TAG, "onNewIntent: intent="+intent+", activity="+this);
        }   
    }

说明：上面的SecondActivity是standard模式的。onBack是按钮的回调函数，点击该按钮，会跳转回ActivityTest。


**测试内容**：ActivityTest --> SecondActivity --> ActivityTest  
**测试结果**：两个ActivityTest是不同的实例！  
**结果分析**：这与我们之前的描述是相符的，当singleTop类型的Activity不在栈顶时，会新建Activity实例。



<a name="anchor4"></a>
# 4. singleTask模式

Google官网对singleTask的描述如下：

> The system creates a new task and instantiates the activity at the root of the new task. However, if an instance of the activity already exists in a separate task, the system routes the intent to the existing instance through a call to its onNewIntent() method, rather than creating a new instance. Only one instance of the activity can exist at a time. 

大致意思是：

> 在singleTask模式下，如果是第一次创建该Activity实例时，则会新建task并将该Activity添加到该task中。否则(该Activity的实例已存在)，则会打开已有的Activity实例，并调用Activity的onNewIntent()方法，而不会新建Activity实例。在任意时刻，最多只会有该一个Activity实例存在。

上面的描述...其实特别抽象。我们通过实例来对singleTask进行了解，最后再对singleTask进行总结。需要建立的一个概念是：singleTask，顾名思义，只容许有一个包含该Activity实例的task存在！


## 4.1 singleTask示例一

点击查看：[singleTask示例一的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/03_single_task/01_same_taskAffinity)

该示例是来验证：(01) 第一次创建singleTask类型的Activity时，会创建新的task！(02) 该Activity实例已经存在时，不会创建新的Activity实例，才是跳转到已有的Activity实例中。

将前面的"singleTop示例二"中的ActivityTest的模式改为"standard"，将SecondActivity的模式改为"singleTask"。修改后的manifest如下：


    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:launchMode="standard"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity"
            android:launchMode="singleTask" />
    </application>


**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity    
**测试结果**：(01) ActivityTest和SecondActivity在同一个task中。 (02) 两个SecondActivity是相同的实例！  
**结果分析**：结论(01)验证失败，结论(02)验证成功!

为什么会出现结论(01)验证失败呢？根据Google官网的描述，分明会启动一个新的task才对啊？为什么呢？  
会不会是由于ActivityTest和SecondActivity位于同一个APK中，由于它们的android:taskAffinity相同导致的！嗯...到底是不是呢？下面就通过示例来进一步验证！


## 4.2 singleTask示例二

点击查看：[singleTask示例二的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/03_single_task/02_diff_taskAffinity)

将"singleTask示例一"中的两个Activity的taskAffinity改为不同，其他保持不变。修改后的manifest如下：

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:launchMode="standard"
                  android:taskAffinity="com.skw.activitytest.task01"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity"
                  android:taskAffinity="com.skw.activitytest.task02"
                  android:launchMode="singleTask" />
    </application>


**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity    
**测试结果**：(01) 第一个ActivityTest和第一个SecondActivity在不同task中。 (02) 两个SecondActivity是相同的实例！  
**结果分析**：结论(01)验证成功，结论(02)验证成功!

进一步进行测试：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity  --> 返回键 --> 返回键     
**测试结果**：在前面测试的基础上，按两次返回键。结果：第一次按返回键的时候，回到第一次创建的ActivityTest实例；再按一次返回键的话，则返回到原始画面(第一次进入ActivityTest之前的画面)！  
**结果分析**：这个结果表明，再次进入SecondActivity时，会将SecondActivity所在task中位于SecondActivity之上的全部Activity都删除！  



总结来说：singleTask的结论与android:taskAffinity相关。以A启动B来说   
(01) 当A和B的taskAffinity相同时：第一次创建B的实例时，并不会启动新的task，而是直接将B添加到A所在的task；否则，将B所在task中位于B之上的全部Activity都删除，然后跳转到B中。  
(02) 当A和B的taskAffinity不同时：第一次创建B的实例时，会启动新的task，然后将B添加到新建的task中；否则，将B所在task中位于B之上的全部Activity都删除，然后跳转到B中。  





<a name="anchor5"></a>
# 5. singleInstance模式

singleInstance，顾名思义，是单一实例的意思，即任意时刻只允许存在唯一的Activity实例！  
根据Google官网的描述，在模式下，只允许有一个Activity实例。当第一次创建该Activity实例时，会新建一个task，并将该Activity添加到该task中。**注意：该task只能容纳该Activity实例，不会再添加其他的Activity实例！**如果该Activity实例已经存在于某个task，则直接跳转到该task。


## 5.1 singleInstance示例一

点击查看：[singleInstance示例一的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/04_single_instance/singleTop_singleInstance)


将前面的"singleTop示例二"中的ActivityTest的模式改为"standard"，将SecondActivity的模式改为"singleInstance"。修改后的manifest如下：

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:launchMode="standard"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity"
                  android:launchMode="singleInstance"/>
    </application>


**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity    
**测试结果**：(01) 两个SecondActivity是同一个实例。 (02) 第一次进入的ActivityTest和第一次进入的SecondActivity位于不同的task中。 (03) 两个ActivityTest是位于同一个task中的不同实例。   
 **结果分析**：这个结论与预期是相同的，即，singleInstance类型的Activity的实例只能有一个，而且它只允许存在于单独的一个task中。singleInstance与相互跳转的两个Activity的taskAffinity无关系！

至于为什么两个ActivityTest是位于同一个task中的不同实例，那是因为它是standard类型的。我们可以将ActivityTest修改为singleTop等其他类型进行测试。



## 5.2 singleInstance示例二

点击查看：[singleInstance示例二的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/activity/launch_mode/04_single_instance/standard_singleInstance)


将前面的"singleInstance示例一"中的ActivityTest的模式改为"singleTop"。修改后的manifest如下：

    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <activity android:name="ActivityTest"
                  android:launchMode="singleTop"
                  android:label="@string/app_name">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity android:name="SecondActivity"
                  android:launchMode="singleInstance"/>
    </application>



**测试内容**：ActivityTest --> SecondActivity --> ActivityTest --> SecondActivity    
**测试结果**：(01) 两个SecondActivity是同一个实例。 (02) 第一次进入的ActivityTest和第一次进入的SecondActivity位于不同的task中。 (03) 两个ActivityTest是同一个实例。   
 **结果分析**：这个结论与预期是相同的。



<a name="anchor6"></a>
# launchMode模式总结


现在，总结一下launchMode的四种模式：

## 1. standard

它是默认模式。在该模式下，Activity可以拥有多个实例，并且这些实例既可以位于同一个task，也可以位于不同的task。  


## 2.singleTop

该模式下，在同一个task中，如果存在该Activity的实例，并且该Activity实例位于栈顶(即，该Activity位于前端)，则调用startActivity()时，不再创建该Activity的示例；而仅仅只是调用Activity的onNewIntent()。否则的话，则新建该Activity的实例，并将其置于栈顶。



## 3. singleTask

顾名思义，只容许有一个包含该Activity实例的task存在！

总的来说：singleTask的结论与android:taskAffinity相关。以A启动B来说   
(01) 当A和B的taskAffinity相同时：第一次创建B的实例时，并不会启动新的task，而是直接将B添加到A所在的task；否则，将B所在task中位于B之上的全部Activity都删除，然后跳转到B中。  
(02) 当A和B的taskAffinity不同时：第一次创建B的实例时，会启动新的task，然后将B添加到新建的task中；否则，将B所在task中位于B之上的全部Activity都删除，然后跳转到B中。  


## 4. singleInstance

顾名思义，是单一实例的意思，即任意时刻只允许存在唯一的Activity实例，而且该Activity所在的task不能容纳除该Activity之外的其他Activity实例！  

它与singleTask有相同之处，也有不同之处。  
**相同之处**：任意时刻，最多只允许存在一个实例。  
**不同之处**：(01) singleTask受android:taskAffinity属性的影响，而singleInstance不受android:taskAffinity的影响。 (02) singleTask所在的task中能有其它的Activity，而singleInstance的task中不能有其他Activity。 (03) 当跳转到singleTask类型的Activity，并且该Activity实例已经存在时，会删除该Activity所在task中位于该Activity之上的全部Activity实例；而跳转到singleInstance类型的Activity，并且该Activity已经存在时，不需要删除其他Activity，因为它所在的task只有该Activity唯一一个Activity实例。

