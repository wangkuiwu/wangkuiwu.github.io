---
layout: post
title: "Android组件--Fragment(二)之 PreferenceFragment"
description: "android training"
category: android
tags: [android]
date: 2014-06-23 09:35
---


> 本文介绍PreferenceFragment。PreferenceFragment是Android提供的设置碎片。


<a name="anchor1"></a>
# PreferenceFragment使用说明

点击查看：[PreferenceFragment的完整代码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/fragment/preference_fragment/01_basic)

## 1. 创建配置文件

新建res/xml/preferences.xml，内容如下：

    <PreferenceScreen xmlns:android="http://schemas.android.com/apk/res/android">

        <PreferenceCategory
            android:title="PreferenceCategory A">

            <!-- 
              (01) android:key是Preferece的id
              (02) android:title是Preferece的大标题
              (03) android:summary是Preferece的小标题
              -->
            <CheckBoxPreference
                android:key="checkbox_preference"
                android:title="title_checkbox_preference"
                android:summary="summary_checkbox_preference" />

        </PreferenceCategory>

        <PreferenceCategory
            android:title="PreferenceCategory B">

            <!-- 
              android:dialogTitle是对话框的标题
              android:defaultValue是默认值
              -->
            <EditTextPreference
                android:key="edittext_preference"
                android:title="title_edittext_preference"
                android:summary="null"  
                android:dialogTitle="dialog_title_edittext_preference"
                android:defaultValue="null" />

            <!-- 
              android:entries是列表中各项的说明
              android:entryValues是列表中各项的值
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


        <PreferenceCategory
            android:title="PreferenceCategory C">

            <SwitchPreference
                android:key="switch_preferece"
                android:title="title_switch_preferece"
                android:defaultValue="true" />

            <SeekBarPreference
                android:key="seekbar_preference"
                android:title="title_seekbar_preference"
                android:max="100"
                android:defaultValue="30" />

        </PreferenceCategory>

    </PreferenceScreen>


说明：PreferenceFragment的组件很多，包括CheckBoxPreference, EditTextPreference, ListPreference, SwitchPreference, SeekBarPreference, VolumePreference等。这些组建的属性定义如下。  
(01) android:key是Preferece的id，它是Preferece的唯一标识。  
(02) android:title是Preferece的大标题。  
(03) android:summary是Preferece的小标题。  
(04) android:dialogTitle是对话框的标题。  
(05) android:defaultValue是默认值。  
(06) android:entries是列表中各项的说明。  
(07) android:entryValues是列表中各项的值。  

注意：SwitchPreference是API 14(Android4.0)才支持的。所以，要想使用SwitchPreference的话，必须在manifest中定义apk支持的最小版本。

    <uses-sdk android:minSdkVersion="14" />


## 2. 自定义PreferenceFragment


    public class PrefsFragment extends PreferenceFragment 
        implements SharedPreferences.OnSharedPreferenceChangeListener, Preference.OnPreferenceClickListener {
        private static final String TAG = "##PrefsFragment##";

        private static final String CHECK_PREFERENCE    = "checkbox_preference";
        private static final String EDITTEXT_PREFERENCE = "edittext_preference";
        private static final String LIST_PREFERENCE     = "list_preference";
        private static final String SWITCH_PREFERENCE   = "switch_preferece";
        private static final String SEEKBAR_PREFERENCE  = "seekbar_preference";

        private Preference mEditText;
        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            // Load the preferences from an XML resource
            addPreferencesFromResource(R.xml.preferences);

            mEditText = (Preference) findPreference(EDITTEXT_PREFERENCE);
            mEditText.setOnPreferenceClickListener(this);
        }

        @Override
        public void onSharedPreferenceChanged(SharedPreferences sharedPreferences, String key) {
            // Set summary to be the user-description for the selected value
            Preference connectionPref = findPreference(key);
            if (key.equals(CHECK_PREFERENCE)) {
                boolean checked = sharedPreferences.getBoolean(key, false);
                Log.d(TAG, "CheckBox: checked="+checked);
            } else if (key.equals(EDITTEXT_PREFERENCE)) {
                String value = sharedPreferences.getString(key, "");
                connectionPref.setSummary(value);
                Log.d(TAG, "EditText: value="+value);
            } else if (key.equals(LIST_PREFERENCE)) {
                String value = sharedPreferences.getString(key, "");
                connectionPref.setSummary(value);
                Log.d(TAG, "List: value="+value);
            } else if (key.equals(SWITCH_PREFERENCE)) {
                boolean checked = sharedPreferences.getBoolean(key, false);
                Log.d(TAG, "Switch: checked="+checked);
            } else if (key.equals(SEEKBAR_PREFERENCE)) {
                int value = sharedPreferences.getInt(key, 0);
                Log.d(TAG, "Seekbar: value="+value);
            } 
        }

        @Override
        public boolean onPreferenceClick(Preference preference) {
            SharedPreferences sharedPreferences = preference.getSharedPreferences();
            String value = sharedPreferences.getString(preference.getKey(), "");
            Log.d(TAG, "onPreferenceClick: value="+value);

            return true;
        }

        @Override
        public void onResume() {
            super.onResume();
            getPreferenceManager().getSharedPreferences().registerOnSharedPreferenceChangeListener(this);

        }

        @Override
        public void onPause() {
            getPreferenceManager().getSharedPreferences().unregisterOnSharedPreferenceChangeListener(this);
            super.onPause();
        }
    }

说明：PreferenceFragment中的每一项都是一个SharedPreferences对象，它们会像SharedPreferences存储在该APK的私有数据区。监听PreferenceFragment中的成员有多种方式，常用的两种就是：  
(01) 监听数据的变化：通过实现SharedPreferences.OnSharedPreferenceChangeListener接口，来监听PreferenceFragment中每一项的数据变化。   
(02) 监听点击事件：通过实现Preference.OnPreferenceClickListener接口，来监听PreferenceFragment中每一项的点击动作。 


## 3. 使用PreferenceFragment

前面已经定义好了一个PreferenceFragment。接下来，就可以实例化它的对象，并将其在Activity中进行显示。


    public class FragmentTest extends Activity {

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            //setContentView(R.layout.main);

            getFragmentManager().beginTransaction().replace(android.R.id.content,  
                    new PrefsFragment()).commit();  
        }
    }

