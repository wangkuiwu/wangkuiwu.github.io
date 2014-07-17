---
layout: post
title: "Android组件--Fragment(三)之 自定义ListPreference"
description: "android training"
category: android
tags: [android]
date: 2014-06-23 09:55
---


> 本文介绍自定义ListPreference的相关内容。


<a name="anchor1"></a>
# ListPreference自定义属性

系统自带的ListPreference的列表中只能显示文本。如果想显示图片或其他内容，只有通过自定义ListPreference的方式。

接下来，将通过示例来演示如何在ListPreference中显示图片。

点击查看：[ListPreference示例的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/fragment/preference_fragment/02_selfdeine_ListPreference_with_attr)

## 1. 自定义属性

添加文件res/values/attrs.xml，内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <resources>
        <declare-styleable name="IconListPreference">
            <attr name="entryIcons" format="reference" />
        </declare-styleable>
    </resources>

说明：  
(01) name="IconListPreference"，与自定义的ListPreference类的名称相对应。后面会实现一个继承于ListPreference的IconListPreference.java。  
(02) name="entryIcons"，这是属性的名称。  
(03) format="reference"，这描述属性的值是引用类型。因为，后面会根据资源id设置该属性，所以将属性格式设为reference。如果是颜色，设为format="color"；如果是布尔类型，format="boolean"；如果是字符串，设为format="string"。  


## 2. 自定义ListPreference


### 2.1 构造函数

    public IconListPreference(Context context, AttributeSet attrs) {
        super(context, attrs);
        mContext = context;

        // 获取自定义的属性(attrs.xml中)对应行的TypedArray
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.IconListPreference);
        // 获取entryIcons属性对应的值
        int iconResId = a.getResourceId(R.styleable.IconListPreference_entryIcons, -1);
        if (iconResId != -1) {
            setEntryIcons(iconResId);
        }   

        // 获取Preferece对应的key
        mKey = getKey();
        // 获取SharedPreferences
        mPref = PreferenceManager.getDefaultSharedPreferences(context);
        // 获取SharedPreferences.Editor
        mEditor = mPref.edit();
        // 获取Entry
        // 注意：如果配置文件中没有android:entries属性，则getEntries()为空；
        mEntries = getEntries();
        // 获取Entry对应的值
        // 注意：如果配置文件中没有android:entryValues属性，则getEntries()为空
        mEntryValues = getEntryValues();

        // 获取该ListPreference保存的值
        String value = mPref.getString(mKey, "");
        mPosition = findIndexOfValue(value);
        // 设置Summary
        if (mPosition!=-1) {
            setSummary(mEntries[mPosition]);
            setIcon(mEntryIcons[mPosition]);
        }   

        a.recycle();
   }   

说明：  
(01) 首先，根据obtainStyledAttributes()能获取自定义属性对应的TypedArray对象。  
(02) 在自定义属性中，entryIcons对应的类名是IconListPreference。因为需要通过"类名"_"属性名"，即IconListPreference_entryIcons的方式来获取资源信息。  
(03) getKey()是获取Preferece对应的Key。该Key是Preference对象的唯一标识。  
(04) getEntries()是获取Preferece的Entry数组。  
(05) getEntryValues()是获取Preferece的Entry对应的值的数组。  
(06) setSummary()是设置Preferece的summary标题内容。  
(07) setIcon()是设置Preferece的图标。  



### 2.2 自定义ListPreference中图片相关代码

    /**
     * 设置图标：icons数组
     */
    private void setEntryIcons(int[] entryIcons) {
        mEntryIcons = entryIcons;
    }

    /**
     * 设置图标：根据icon的id数组
     */
    public void setEntryIcons(int entryIconsResId) {
        TypedArray icons = getContext().getResources().obtainTypedArray(entryIconsResId);
        int[] ids = new int[icons.length()];
        for (int i = 0; i < icons.length(); i++)
            ids[i] = icons.getResourceId(i, -1);
        setEntryIcons(ids);
        icons.recycle();
    }

说明：这两个函数是读取图片信息的。


### 2.3 自定义ListPreference弹出的列表选项

    @Override
    protected void onPrepareDialogBuilder(Builder builder) {
        super.onPrepareDialogBuilder(builder);

        IconAdapter adapter = new IconAdapter(mContext);
        builder.setAdapter(adapter, null);
    }

说明：点击ListPreference，会弹出一个列表对话框。通过重写onPrepareDialogBuilder()，我们可以自定义弹出的列表对话框。这里是通过IconAdapter来显示的。


    public class IconAdapter extends BaseAdapter{

        private LayoutInflater mInflater;


        public IconAdapter(Context context){
            this.mInflater = LayoutInflater.from(context);
        }
        @Override
        public int getCount() {
            return mEntryIcons.length;
        }

        @Override
        public Object getItem(int arg0) {
            return null;
        }

        @Override
        public long getItemId(int arg0) {
            return 0;
        }

        @Override
        public View getView(int position, View convertView, ViewGroup parent) {

            ViewHolder holder = null;
            if (convertView == null) {

                holder = new ViewHolder();

                convertView = mInflater.inflate(R.layout.icon_adapter, parent, false);
                holder.layout = (LinearLayout)convertView.findViewById(R.id.icon_layout);
                holder.img = (ImageView)convertView.findViewById(R.id.icon_img);
                holder.info = (TextView)convertView.findViewById(R.id.icon_info);
                holder.check = (RadioButton)convertView.findViewById(R.id.icon_check);
                convertView.setTag(holder);

            }else {
                holder = (ViewHolder)convertView.getTag();
            }

            holder.img.setBackgroundResource(mEntryIcons[position]);
            holder.info.setText(mEntries[position]);
            holder.check.setChecked(mPosition == position);

            final ViewHolder fholder = holder;
            final int fpos = position;
            convertView.setOnClickListener(new View.OnClickListener() {

                @Override
                public void onClick(View v) {
                    v.requestFocus();
                    // 选中效果
                    fholder.layout.setBackgroundColor(Color.CYAN);

                    // 更新mPosition
                    mPosition = fpos;
                    // 更新Summary
                    IconListPreference.this.setSummary(mEntries[fpos]);
                    IconListPreference.this.setIcon(mEntryIcons[fpos]);
                    // 更新该ListPreference保存的值
                    mEditor.putString(mKey, mEntryValues[fpos].toString());
                    mEditor.commit();

                    // 取消ListPreference设置对话框
                    getDialog().dismiss();
                }
            });

            return convertView;
        }

        // ListPreference每一项对应的Layout文件的结构体
        private final class ViewHolder {
            ImageView img;
            TextView info;
            RadioButton check;
            LinearLayout layout;
        }
    }

说明：弹出的列表对话框中的每一项的内容是通过布局icon_adapter.xml来显示的。下面看看icon_adapter.xml的源码。

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/icon_layout" 
        android:orientation="horizontal"
        android:paddingLeft="6dp"  
        android:layout_width="fill_parent"
        android:layout_height="fill_parent">
      
      
        <ImageView
            android:id="@+id/icon_img" 
            android:layout_width="wrap_content"
            android:layout_height="wrap_content" 
            android:gravity="center_vertical"
            android:layout_margin="4dp"/>
      
        <TextView
            android:id="@+id/icon_info" 
            android:layout_width="0dp"
            android:layout_height="wrap_content" 
            android:layout_weight="1"
            android:paddingLeft="6dp"
            android:layout_gravity="left|center_vertical"
            android:textAppearance="?android:attr/textAppearanceLarge" />
              
        <RadioButton
            android:id="@+id/icon_check"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:checked="false"
            android:layout_gravity="right|center_vertical"
            android:layout_marginRight="6dp"/>
      
    </LinearLayout>

至此，自定义的ListPreference就算完成了。下面就是如何使用它了。



## 3. 使用该自定义ListPreference

我们是通过PreferenceFragment使用该自定义的ListPreference。

### 3.1 PreferenceFragment的配置文件

res/xml/preferences.xml的内容如下：

    <PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:iconlistpreference="http://schemas.android.com/apk/res/com.skw.fragmenttest">
        
        <!-- 系统默认的ListPreference -->
        <PreferenceCategory
            android:title="PreferenceCategory A">

            <!-- 
              (01) android:key是Preferece的id
              (02) android:title是Preferece的大标题
              (03) android:summary是Preferece的小标题
              (04) android:dialogTitle是对话框的标题
              (05) android:defaultValue是默认值
              (06) android:entries是列表中各项的说明
              (07) android:entryValues是列表中各项的值
              -->
            <ListPreference  
                android:key="list_preference"  
                android:dialogTitle="Choose font"  
                android:entries="@array/pref_font_types"  
                android:entryValues="@array/pref_font_types_values"  
                android:summary="sans"  
                android:title="Font" 
                android:defaultValue="sans"/> 
        </PreferenceCategory>

        <!-- 自定义的ListPreference -->

        <PreferenceCategory
            android:title="PreferenceCategory B">

            <!-- 
              iconlistpreference:entryIcons是自定义的属性
              -->
            <com.skw.fragmenttest.IconListPreference
                android:key="icon_list_preference"  
                android:dialogTitle="ChooseIcon"  
                android:entries="@array/android_versions"
                android:entryValues="@array/android_version_values"  
                iconlistpreference:entryIcons="@array/android_version_icons"
                android:icon="@drawable/cupcake"
                android:summary="summary_icon_list_preference"
                android:title="title_icon_list_preference" /> 

        </PreferenceCategory>

    </PreferenceScreen>

说明：该配置文件中使用了"系统默认的ListPreference"和"自定义的ListPreference(即IconListPreference)"。  
注意，**IconListPreference中的"iconlistpreference:entryIcons"属性。前面的"iconlistpreference"与该文件的命名空间表示"xmlns:iconlistpreference="http://schemas.android.com/apk/res/com.skw.fragmenttest"中的iconlistpreference一样! 而entryIcons则是我们自定义的属性名称**。 
        


### 3.2 自定义PreferenceFragment的代码

    public class PrefsFragment extends PreferenceFragment {

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            addPreferencesFromResource(R.xml.preferences);
        }

        ...
    }


## 4. 使用PrefsFragment

下面，就可以在Activity中使用该PrefsFragment了。

## 4.1 使用PrefsFragment的Activity的代码

    public class FragmentTest extends Activity {

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            // 获取FragmentManager
            FragmentManager fragmentManager = getFragmentManager();
            // 获取FragmentTransaction        
            FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
                    
            PrefsFragment fragment = new PrefsFragment();
            // 将fragment添加到容器frag_example中
            fragmentTransaction.add(R.id.prefs, fragment);
            fragmentTransaction.commit();
        }   
    }


## 4.2 使用PrefsFragment的Activity的配置文件

res/layout/main.xml的内容如下：

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:orientation="vertical"
        android:layout_width="fill_parent"
        android:layout_height="fill_parent"
        >
      
        <FrameLayout
            android:id="@+id/prefs"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
      
    </LinearLayout>



<a name="anchor1"></a>
# ListPreference自定义说明

如果你想通过提供API的方式，而不是配置属性的方式完成上面的工作。那么，也是可以办到的！

点击查看：[修改后的自定义ListPreference源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/fragment/preference_fragment/03_selfdeine_ListPreference_plus_api)

**不过，还是建议采用配置属性的方式！**



