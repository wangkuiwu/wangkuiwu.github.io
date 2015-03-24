---
layout: post
title: "Android控件篇17之 DatePicker和TimePicker"
description: "android training"
category: android
tags: [android]
date: 2014-02-17 09:11
---



<a name="anchor1"></a>
# 1. Picker简介

DatePicker和TimePicker分别提供日期和时间的选择试图；通过它们得到的日期和时间是格式化的。



<a name="anchor2"></a>
# 2. Picker示例

写一个activity，包含一个“日期”按钮和一个“时间”按钮。  
点击“日期”按钮，进入“日期”选择界面。  
点击“时间”按钮，进入“时间”选择界面。

**应用层代码**

    public class PickerTest extends FragmentActivity {
        private static final String TAG = "SKYWANG";

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.picker_test);
        }
        
        // "时间"按钮onClick
        public void showTimePickerDialog(View v) {
            DialogFragment newFragment = new TimePickerFragment();
            newFragment.show(getSupportFragmentManager(), "timePicker");
        }
        
        public static class TimePickerFragment extends DialogFragment
            implements TimePickerDialog.OnTimeSetListener {
        
            @Override
            public Dialog onCreateDialog(Bundle savedInstanceState) {
                // Use the current time as the default values for the picker
                final Calendar c = Calendar.getInstance();
                int hour = c.get(Calendar.HOUR_OF_DAY);
                int minute = c.get(Calendar.MINUTE);
                
                // Create a new instance of TimePickerDialog and return it
                return new TimePickerDialog(getActivity(), this, hour, minute,
                        DateFormat.is24HourFormat(getActivity()));
            }
            
            public void onTimeSet(TimePicker view, int hourOfDay, int minute) {
                Log.d(TAG, "hour:"+hourOfDay+", minute: "+minute);
            }
        }
        
        // “日期”按钮onClick
        public void showDatePickerDialog(View v) {
            DialogFragment newFragment = new DatePickerFragment();
            newFragment.show(getSupportFragmentManager(), "datePicker");
        }
        
        public static class DatePickerFragment extends DialogFragment
            implements DatePickerDialog.OnDateSetListener {
        
            @Override
            public Dialog onCreateDialog(Bundle savedInstanceState) {
                // Use the current date as the default date in the picker
                final Calendar c = Calendar.getInstance();
                int year = c.get(Calendar.YEAR);
                int month = c.get(Calendar.MONTH);
                int day = c.get(Calendar.DAY_OF_MONTH);
                
                // Create a new instance of DatePickerDialog and return it
                return new DatePickerDialog(getActivity(), this, year, month, day);
            }
            
            public void onDateSet(DatePicker view, int year, int month, int day) {
                Log.d(TAG, "year:"+year+", month: "+month+", day:"+day);
            }
        }
            
    }

**layout文件**

    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >
        
        <Button 
        android:layout_width="wrap_content" 
        android:layout_height="wrap_content"
        android:text="@string/pick_time" 
        android:onClick="showTimePickerDialog" />

        <Button 
        android:layout_width="wrap_content" 
        android:layout_height="wrap_content"
        android:text="@string/pick_date" 
        android:onClick="showDatePickerDialog" />
        
    </LinearLayout>

运行效果如下图

![img](/media/pic/android/widgets/datetime01.jpg)
 

日期选择窗口

![img](/media/pic/android/widgets/datetime02.jpg)
 

时间选择窗口

![img](/media/pic/android/widgets/datetime03.jpg)

