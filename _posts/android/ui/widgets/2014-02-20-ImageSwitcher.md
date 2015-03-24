---
layout: post
title: "Android控件篇20之 ImageSwticher"
description: "android training"
category: android
tags: [android]
date: 2014-02-20 09:11
---
 

 

<a name="anchor1"></a>
# 1. ImageSwticher介绍

ImageSwitcher是图片切换的控件，它能实现图片切换时的动画效果，包括图片导入效果、图片消失效果等等。Android系统提供了许多不同的动画效果供我们选择。

 

 
<a name="anchor2"></a>
# 2. 应用示例

示例说明：新建一个activity，包括一个ImageSwitcher控件。ImageSwitcher中的图片，每5秒钟变换一个。

代码说明：

    public class ImageSwitcherTest extends Activity implements ViewFactory {
        
        private ImageSwitcher mSwitcher;

        // 图片数组，用于在ImageSwitcher中切换
        private Integer[] mImageIds = { R.drawable.pic1,
                R.drawable.pic2, R.drawable.pic3, R.drawable.pic4 };
        // 图片显示序号
        private int mIndex = 0;
        private ImageSwitchThread mImgThread;
        
        private static final int MSG_SWITCH_IMG = 0x00000001;
        private Handler mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MSG_SWITCH_IMG: {
                        mIndex = (mIndex + 1) % mImageIds.length;
                        mSwitcher.setImageResource(mImageIds[mIndex]);
                        break;    
                    }
                    default:
                        break;
                }
            }
        };
        
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            requestWindowFeature(Window.FEATURE_NO_TITLE);
            setContentView(R.layout.image_switcher_test);        
            setTitle("ImageShowActivity");

            mIndex = 0;
            mSwitcher = (ImageSwitcher) findViewById(R.id.ImageSwitcher01);
            // setFactory的具体实现参考makeView
            mSwitcher.setFactory(this);    
            // 设置ImageSwitcher移入图片时的动画效果
            mSwitcher.setInAnimation(AnimationUtils.loadAnimation(this,
                    android.R.anim.slide_in_left));
            // 设置ImageSwitcher移除图片时的动画效果
            mSwitcher.setOutAnimation(AnimationUtils.loadAnimation(this,
                    android.R.anim.slide_out_right));
            
            mImgThread = new ImageSwitchThread();        
        }

        @Override
        protected void onResume() {
            super.onResume();
            if (mImgThread != null)
                mImgThread.start();
        }
        @Override
        protected void onPause() {
            super.onPause();
            if (mImgThread != null)
                mImgThread.interrupt();
        }
        
        @Override
        public void onDestroy() {
            super.onDestroy();
        }

        /*
         * makeView()返回的view将被添加到ImageSwitcher中。
         * 也就意味着，第一次若没有设置ImageSwitcher的背景，makeView()的返回值，将作为默认背景。
         * @author skywang
         */
        @Override
        public View makeView() {
            ImageView i = new ImageView(this);
            i.setBackgroundColor(0xFF000000);
            i.setScaleType(ImageView.ScaleType.FIT_CENTER);
            i.setLayoutParams(new ImageSwitcher.LayoutParams(
                    LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT));
            return i;
        }

        /**
         * 图片更新线程。每5秒更新一次图片
         * @author skywang
         *
         */
        private class ImageSwitchThread extends Thread {
            @Override
            public void run() {
                super.run();
                try {
                    while (!isInterrupted()) {
                        // 发送消息给mHanlder更换图片
                        mHandler.sendEmptyMessage(MSG_SWITCH_IMG);
                        Thread.sleep(5000);
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

说明:  
(01) mSwitcher.setFactory(this); setFactory()的作用是设置ImageSwitcher切换时，用于填充的两个视图。具体的实现是通过makeView()来实现的。  
(02) mSwitcher.setInAnimation(AnimationUtils.loadAnimation(this, android.R.anim.slide_in_left)); 这是来设置ImageSwitcher显示图片时的动画效果，android.R.anim.slide_in_left是android自带的动画效果，即图片显示时，从左边逐渐显示。  
(03) mSwitcher.setOutAnimation(AnimationUtils.loadAnimation(this, android.R.anim.slide_out_right)); 这是来设置ImageSwitcher图片消失时的动画效果，android.R.anim.slide_out_right是android自带的动画效果，即图片消失时，从右边逐渐消失。  
(04) mImgThread = new ImageSwitchThread(); 用于创建线程，线程ImageSwitchThread的作用是每隔5秒发送消息来更新ImageSwitcher中的图片。


运行效果如下图

![img](/media/pic/android/widgets/imageswitcher01.jpg)
 
