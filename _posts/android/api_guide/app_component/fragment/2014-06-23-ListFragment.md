---
layout: post
title: "Android组件--Fragment(四)之 ListFragment"
description: "android training"
category: android
tags: [android]
date: 2014-06-23 09:45
---


> 本文介绍ListFragment。


<a name="anchor1"></a>
# ListFragment简介

ListFragment继承于Fragment。因此它具有Fragment的特性，能够作为activity中的一部分，目的也是为了使页面设计更加灵活。

相比Fragment，ListFragment的内容是以列表(list)的形式显示的。ListFragment的布局默认包含一个ListView。因此，在ListFragment对应的布局文件中，必须指定一个 android:id 为 “@android:id/list” 的ListView控件! 


# ListFragment使用示例

点击查看：[ListFragment的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/fragment/list_fragment/01_simple)

下面介绍在Activity中显示ListFragment的步骤。

## 1. Activity对应的代码

    public class FragmentTest extends Activity {
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);
        }   
    }


## 2. Activity对应的布局

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="horizontal" >

        <fragment 
            android:name="com.skw.fragmenttest.MyListFragment"
            android:id="@+id/myfragment"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </LinearLayout>

说明：该Activity的布局中只包行了一个Fragment。下面看看MyListFragment的内容。


## 3. MyListFragment的内容

    public class MyListFragment extends ListFragment {
        private static final String TAG = "##MyListFragment##";

        private ListView selfList;

        String[] cities = {
             "Shenzhen",
             "Beijing",
             "Shanghai",
             "Guangzhou",
             "Wuhan",
             "Tianjing",
             "Changsha",
             "Xi'an",
             "Chongqing",
             "Guilin",
        };

        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container, 
                Bundle savedInstanceState) {
            Log.d(TAG, "onCreateView");
            return inflater.inflate(R.layout.list_fragment, container, false);
        }
        

        @Override
        public void onCreate(Bundle savedInstanceState) {
            Log.d(TAG, "onCreate");
            super.onCreate(savedInstanceState);
            // 设置ListFragment默认的ListView，即@id/android:list
            this.setListAdapter(new ArrayAdapter<String>(getActivity(), 
                    android.R.layout.simple_list_item_1, cities));
            
        }
       
        public void onListItemClick(ListView parent, View v, 
                int position, long id) {
            Log.d(TAG, "onListItemClick");
            Toast.makeText(getActivity(), "You have selected " + cities[position],
                    Toast.LENGTH_SHORT).show();
        }    
    }

说明：MyListFragment是自定义的ListFragment。它使用了list_fragment.xml作为布局，并通过android.R.layout.simple_list_item_1显示ListView中的每一项。


## 4. list_fragment.xml的内容

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >
              
        <!-- ListFragment对应的android:id值固定为"@id/android:list" -->
        <ListView
            android:id="@id/android:list"
            android:layout_width="match_parent"
            android:layout_height="match_parent" 
            android:drawSelectorOnTop="false"
            />
            
    </LinearLayout>





# 自定义ListFragment

点击查看：[自定义ListFragment的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/fragment/list_fragment/02_self_layout)


"Activity的布局以及代码"和前面一样，这里就不再重复说明。


## 3. MyListFragment的内容


    public class MyListFragment extends ListFragment {
        private static final String TAG = "##MyListFragment##";
        
        private ListView selfList;
        
        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container, 
                Bundle savedInstanceState) {
            Log.d(TAG, "onCreateView");
            return inflater.inflate(R.layout.list_fragment, container, false);
        }
        
        @Override
        public void onCreate(Bundle savedInstanceState) {
            final String[] from = new String[] {"title", "info"};
            final int[] to = new int[] {R.id.text1, R.id.text2};
            
            Log.d(TAG, "onCreate");
            super.onCreate(savedInstanceState);
            // 建立SimpleAdapter，将from和to对应起来
            SimpleAdapter adapter = new SimpleAdapter(
                    this.getActivity(), getSimpleData(), 
                    R.layout.item, from, to);
            this.setListAdapter(adapter);
        }
       
        public void onListItemClick(ListView parent, View v, 
                int position, long id) {
            Log.d(TAG, "onListItemClick");
            Toast.makeText(getActivity(), 
                    "You have selected " + position,
                    Toast.LENGTH_SHORT).show();
        }
        
        private List<Map<String, Object>> getSimpleData() {
            List<Map<String, Object>> list = new ArrayList<Map<String, Object>>();
            
            Map<String, Object> map = new HashMap<String, Object>();
            map.put("title", "Ferris wheel");
            map.put("info", "Suzhou Ferris wheel");
            list.add(map);

            map = new HashMap<String, Object>();
            map.put("title", "Flower");
            map.put("info", "Roser");
            list.add(map);

            map = new HashMap<String, Object>();
            map.put("title", "Disk");
            map.put("info", "Song Disk");
            list.add(map);
            
            return list;
        }
    }


说明：MyListFragment使用了R.layout.list_fragment作为布局，并且对于ListView中的每一项都使用了R.layout.item作为布局。


## 4. list_fragment.xml的内容

    <!-- ListFragment对应的android:id值固定为"@id/android:list" -->
    <ListView
        android:id="@id/android:list"
        android:layout_width="match_parent"
        android:layout_height="match_parent" 
        android:drawSelectorOnTop="false"
        />
    

## 5. item.xml的内容

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >
        
        <TextView android:id="@+id/text1"
            android:textSize="12sp"
            android:textStyle="bold"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"/>

        <TextView android:id="@+id/text2"
            android:textSize="24sp"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"/>
            
    </LinearLayout>




