---
layout: post
title: "Android控件篇14之 ProgressBar"
description: "android training"
category: android
tags: [android]
date: 2014-02-14 09:11
---



<a name="anchor1"></a>
# 1. ProgressBar简介

ProgressBar是进度条，常用于显示程序加载/安装进度等。


<a name="anchor2"></a>
# 2. ProgressBar示例

创建一个activity，包含1个ProgressBar。在Activity中开启一个线程，线程不断的增加ProgressBar的进度；当进度增加满的时候，隐藏ProgressBar。

**应用层代码**

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

**layout文件**

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >
        
       <!-- 
        ProgressBar的style可以如下：
        Widget.ProgressBar.Horizontal
        Widget.ProgressBar.Small
        Widget.ProgressBar.Large
        Widget.ProgressBar.Inverse
        Widget.ProgressBar.Small.Inverse
        Widget.ProgressBar.Large.Inverse   
        -->
        <ProgressBar
            android:id="@+id/pbar_def"
            android:layout_width="600px"
            android:layout_height="wrap_content"
            style="@android:style/Widget.ProgressBar.Horizontal"
            android:max="100"
            android:progress="10"
        />


    </LinearLayout>

style="@android:style/Widget.ProgressBar.Horizontal"  
style的值可以为以下任意一个值：

    Widget.ProgressBar.Horizontal
    Widget.ProgressBar.Small
    Widget.ProgressBar.Large
    Widget.ProgressBar.Inverse
    Widget.ProgressBar.Small.Inverse
    Widget.ProgressBar.Large.Inverse


运行效果如下图

![img](/media/pic/android/widgets/progressbar01.jpg)
 
