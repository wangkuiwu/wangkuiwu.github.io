---
layout: post
title: "Android控件篇22之 ViewPager"
description: "android training"
category: android
tags: [android]
date: 2014-02-22 09:13
---


> 本文介绍ViewPager


<a name="anchor1"></a>
# 1. ViewPager的简介

ViewPager是ViewGroup的子类。和其他ViewGroup一样，ViewPager中能容纳多个View。此外，ViewPager的特点：  
(01) 每次只能显示一个View。  
(02) 不同的View之间能通过左右滑动来进行View的切换。

使用ViewPager和使用ListView等视图有点类似，它们都是通过Adapter来显示的。ViewPager是通过PageAdapter来管理各个View。关于PageAdapter有几点需要强调的：  
(01) PageAdapter是个抽象类。通常我们使用ViewPager时，都需要自定义PageAdapter。在自定义PageAdapter时，一定要实现getCount()和isViewFromObject(view, object)这两个方法。  
(02) 如果ViewPager中的View采用Fragment的话，在自定义PageAdapter时，可以使用PageAdapter与Fragment相关的子类FragmentPagerAdapter或FragmentStatePagerAdapter等。



<a name="anchor2"></a>
# 2. ViewPager示例一

该示例将演示ViewPager的最基本的用法。示例中会创建一个Activity，该Activity的布局中包含一个ViewPager；我们将三个xml布局通过LayoutInflater解析得到View，然后再将这些View添加到该ViewPager对应的PageAdapter中，从而达到使这三个布局能够通过ViewPager相互切换的目的。

点击查看：[ViewPager示例一的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/viewpager/01_basic/ViewPageTest)


## 2.1 默认Activity的布局

默认的Activity的布局文件res/layout/main.xml的内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        >

        <android.support.v4.view.ViewPager
            android:id="@+id/viewpager"
            android:layout_width="fill_parent"
            android:layout_height="fill_parent" />
      
    </LinearLayout>

说明：布局中只包含了ViewPager一个视图。




## 2.2 默认Activity的代码

    public class ViewPageTest extends Activity {
        /** Called when the activity is first created. */
        
        private ViewPager mViewPager;
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            initViews() ;
        }

        private void initViews() {
            LayoutInflater inflater = LayoutInflater.from(this);

            List<View> views = new ArrayList<View>();
            // 初始化引导图片列表
            views.add(inflater.inflate(R.layout.page_one, null));
            views.add(inflater.inflate(R.layout.page_two, null));
            views.add(inflater.inflate(R.layout.page_three, null));

            // 初始化Adapter
            PagerAdapter adapter = new MyPagerAdapter(views);

            mViewPager = (ViewPager) findViewById(R.id.viewpager);
            mViewPager.setAdapter(adapter);
        }

        ...
    }

说明：page_one.xml,page_two.xml和page_three.xml是三个布局文件。它们的内容类似，下面给出page_one.xml的代码。


res/layout/page_one.xml的内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        android:background="#FFAA00"
        >

        <TextView
            android:layout_gravity="center"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Page One"
            />
      
    </LinearLayout>



## 2.3 自定义PageAdapter

在自定义PageAdapter时，一定要实现getCount()和isViewFromObject(view, object)这两个方法。

    private class MyPagerAdapter extends PagerAdapter {

        // 界面列表
        private List<View> views;

        public MyPagerAdapter(List<View> views) {
            this.views = views;
        }   

        // 销毁arg1位置的界面
        @Override
        public void destroyItem(View arg0, int arg1, Object arg2) {
            ((ViewPager) arg0).removeView(views.get(arg1));
        }

        // 获得当前界面数
        @Override
        public int getCount() {
            return views.size();
        }

        // 判断是否由对象生成界面
        @Override
        public boolean isViewFromObject(View arg0, Object arg1) {
            return (arg0 == arg1);
        }

        // 初始化arg1位置的界面
        @Override
        public Object instantiateItem(View arg0, int arg1) {
            ((ViewPager) arg0).addView(views.get(arg1), 0);
            return views.get(arg1);
        }
    }



<a name="anchor3"></a>
# 3. ViewPager示例二

前面的ViewPage中的每一个View都是直接使用的layout布局文件。本示例将演示使用Fragment。

点击查看：[ViewPager示例二的源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/ui/viewpager/02_fragment_viewpager/ViewPageTest)

## 3.1 默认Activity的布局


    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        >

        <android.support.v4.view.ViewPager
            android:id="@+id/viewpager"
            android:layout_width="fill_parent"
            android:layout_height="fill_parent" />
      
    </LinearLayout>

说明：布局和"示例一"中的一样，只包含了ViewPager一个视图。



## 3.2 默认Activity的代码


    public class ViewPageTest extends FragmentActivity {
            
        private ViewPager mViewPager;
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            initViews() ;
        }   

        private void initViews() {
            LayoutInflater inflater = LayoutInflater.from(this);

            List<Fragment> flist = new ArrayList<Fragment>();
            // 初始化Fragment
            flist.add(MyFragment.newInstance("Page 1"));
            flist.add(MyFragment.newInstance("Page 2"));
            flist.add(MyFragment.newInstance("Page 3"));

            // 初始化Adapter
            PagerAdapter adapter = new MyPagerAdapter(getSupportFragmentManager(), flist);

            mViewPager = (ViewPager) findViewById(R.id.viewpager);
            mViewPager.setAdapter(adapter);
        }   
        
        ...
    }

说明：本示例中添加到PagerAdapter的视图都是Fragment对象。**这是本示例与"示例一"的本质区别**！


## 3.3 自定义的Fragment

    public class MyFragment extends Fragment {

        private static final String EXTRA_MESSAGE = "EXTRA_MESSAGE";
                
        public static final MyFragment newInstance(String message) {
            MyFragment f = new MyFragment();
            Bundle bundle = new Bundle(1);
            bundle.putString(EXTRA_MESSAGE, message);
            f.setArguments(bundle);
            return f;
        }   

        @Override
        public View onCreateView(LayoutInflater inflater, 
                ViewGroup container, Bundle savedInstanceState) {
            String message = getArguments().getString(EXTRA_MESSAGE);
            View view = inflater.inflate(R.layout.myfragment, container, false);
            TextView messageTextView = (TextView)view.findViewById(R.id.tv);
            messageTextView.setText(message);
                    
            return view;
        }   
    }

说明：MyFragment的内容非常简单。在通过newInstance()创建该Fragment时，它会保存字符串；然后在显示的时候，再将该字符串赋值给TextView。

MyFragment的布局文件myfragment.xml内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        >

        <TextView
            android:id="@+id/tv"
            android:layout_gravity="center"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            />
      
    </LinearLayout>


## 3.4 自定义PageAdapter


    private class MyPagerAdapter extends FragmentPagerAdapter {
        private List<Fragment> fragments;

        public MyPagerAdapter(FragmentManager fm, List<Fragment> fragments) {
            super(fm);
            this.fragments = fragments;
        }
        @Override
        public Fragment getItem(int position) {
            return this.fragments.get(position);
        }

        @Override
        public int getCount() {
            return this.fragments.size();
        }
    }

说明：本例中MyPagerAdapter的父类是FragmentPagerAdapter，而不是PageAdapter！





