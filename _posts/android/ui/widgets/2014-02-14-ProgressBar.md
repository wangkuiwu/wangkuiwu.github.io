---
layout: post
title: "Android控件篇14之 ProgressBar"
description: "android training"
category: android
tags: [android]
date: 2014-02-14 09:11
---

> 本章介绍ProgressBar进度条。内容包括：条形进度条、圆形进度条和自定义图形。

> 点击查看：[ProgressBar示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/widgets/ProgressBar/01_basic/ProBarTest)

> **目录**  
[1. 条形进度条](#anchor1)  
[2. 圆形进度条](#anchor2)  
[3. 自定义图形](#anchor3)  


<a name="anchor1"></a>
# 1. 条形进度条

ProgressBar是进度条，常用于显示程序加载/安装进度等。

下面以一则示例的方式演示ProgressBar进度条的加载。

## 示例说明
创建一个activity，包含1个ProgressBar。在Activity中开启一个线程，线程不断的增加ProgressBar的进度；当进度增加满的时候，隐藏ProgressBar。


## layout文件

    <ProgressBar
        android:id="@+id/pbar_def"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        style="@android:style/Widget.ProgressBar.Horizontal"
        android:max="100"
        android:progress="10" />

style的值可以为以下任意一个值：

    Widget.ProgressBar.Horizontal
    Widget.ProgressBar.Small
    Widget.ProgressBar.Large
    Widget.ProgressBar.Inverse
    Widget.ProgressBar.Small.Inverse
    Widget.ProgressBar.Large.Inverse


## 应用层代码

    public class ProcessBarTest extends Activity {
        private static final String TAG="SKYWANG";

        // 计数线程
        private CountThread mCountThread;
        // ProcessBar
        private ProgressBar mProgressBar;
        
        // 处理当ProcessBar的进度完成的情况
        private static final int MSG_PROGRESS_BAR_FULL = 1; 
        private Handler handler = new Handler() {

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                case MSG_PROGRESS_BAR_FULL: {
                    Log.d(TAG, "make the ProgressBar Gone!");
                    // 隐藏ProcessBar。
                    mProgressBar.setVisibility(View.GONE);
                    // 终止线程
                    if (mCountThread != null)
                        mCountThread.interrupt();
                        break;
                    }
                default:
                    break;
                }
            }
        };
        
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.process_bar_test);

            mProgressBar = (ProgressBar) findViewById(R.id.pbar_def);
            
            // 开启计数线程
            mCountThread = new CountThread();
            mCountThread.start();
        }

        private class CountThread extends Thread {

            @Override
            public void run() {
                super.run();

                while (!isInterrupted()) {
                    // 线程休眠200ms
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    
                    int progress = mProgressBar.getProgress() + 2;
                    int max = mProgressBar.getMax();

                    Log.d(TAG, "progress : "+progress+" , max : "+max);
                    if (progress < max)
                        mProgressBar.setProgress(progress);
                    else {
                        mProgressBar.setProgress(max);
                        // 发送消息给handler，让它隐藏ProcessBar。
                        handler.sendEmptyMessage(MSG_PROGRESS_BAR_FULL);
                    }
                }   
            }
        }
        
        @Override
        public void onDestroy() {
            super.onDestroy();
            // 终止计数线程
            if (mCountThread != null)
                mCountThread.interrupt();
        }

    }

代码说明：  
CountThread是ProcessBarTest的内部线程类。  
在打开ProcessBarTest时，会自动执行onCreate()；然后通过new CountThread()新建线程，再mCountThread.start()启动线程。  
线程启动后，每个200ms将进度条的进度值+2；然后进度条已经满格，则调用handler.sendEmptyMessage(MSG_PROGRESS_BAR_FULL)发送消息。MSG_PROGRESS_BAR_FULL消息会被主线程的Handler函数捕获并处理：隐藏“进度条”，然后终止线程。

运行效果如下图

![img](/media/pic/android/widgets/progressbar01.jpg)
 



<a name="anchor2"></a>
# 2. 圆形进度条

除了上面的条形进度条之外，ProgressBar还可以以圆形进度条的方式存在。

下面以一则示例的方式演示圆形进度条。

## 示例说明

显示一个圆形进度条，进度条无限加载。


## layout文件

    <ProgressBar
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:indeterminate="true"
        android:indeterminateDrawable="@drawable/bg_progress_infinite" />

bg_progress_infinite.xml的内容如下：

    <?xml version="1.0" encoding="utf-8"?>  
    <!-- 旋转动作：旋转中心点是矩形中心。从0-360度进行旋转 -->
    <rotate xmlns:android="http://schemas.android.com/apk/res/android"
        android:pivotX="50%" 
        android:pivotY="50%" 
        android:fromDegrees="0"
        android:toDegrees="360">

        <!-- ring表示是圆环类型
             innerRadiusRatio=3表示"内圆环的半径=圆环的宽度/3"
             thicknessRatio=8表示"内圆环的厚度=圆环的宽度/8"
        -->
        <shape android:shape="ring" 
            android:innerRadiusRatio="3"
            android:thicknessRatio="10"
            android:useLevel="false">

            <!-- 扫描渐变 -->
            <gradient 
                android:type="sweep"
                android:useLevel="false" 
                android:startColor="#222222"
                android:centerColor="#666666"
                android:endColor="#ffffff"
                android:centerX="0.5"
                android:centerY="0.5" />  
        </shape>  
    </rotate>  

至于shape和selector的更多说明，可以参考文章：[Button样式][link_android_widgets_button02]



<a name="anchor3"></a>
# 3. 自定义图形

还可以对ProgressBar进行自定义图形

## layout文件

    <ProgressBar
        android:id="@+id/pbar_def"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:indeterminate="false"
        android:indeterminateOnly="true"
        android:indeterminateDrawable="@drawable/bg_progress_pic"
        android:indeterminateDuration="3000" />

其中，bg_progress_pic.xml的内容如下：

    <?xml version="1.0" encoding="utf-8"?>  
    <layer-list xmlns:android="http://schemas.android.com/apk/res/android">  
        <item>
            <rotate android:drawable="@drawable/ic_launcher"
                android:fromDegrees="-90.0" android:toDegrees="90.0" 
                android:pivotX="50.0%" android:pivotY="50.0%" />
        </item>  
    </layer-list> 


[link_android_widgets_button02]:   /2014/02/03/Button02
