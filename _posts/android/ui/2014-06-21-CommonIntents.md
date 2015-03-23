---
layout: post
title: "Android API指南(二)Intent之 常用Intent"
description: "android training"
category: android
tags: [android]
date: 2014-06-21 09:11
---


> 在[Android培训篇之App交互类Intent][link_intent_introduce]中已经详细介绍过Intent的基础知识。这里就不再对Intent进行介绍，而是重点介绍系统自带的常用Intent。


<a name="anchor1"></a>
# 1. 闹钟和秒表

点击查看：[闹钟和秒表的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/01_alarm)

启动闹钟的Intent示意如下：

    public void createAlarm(String message, int hour, int minutes) {

        Intent intent = new Intent(AlarmClock.ACTION_SET_ALARM)
                .putExtra(AlarmClock.EXTRA_MESSAGE, message)
                .putExtra(AlarmClock.EXTRA_HOUR, hour)
                .putExtra(AlarmClock.EXTRA_MINUTES, minutes);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivity(intent);
        }   
    }   

启动秒表的Intent示意如下：

    public void startTimer(String message, int seconds) {
        Intent intent = new Intent(AlarmClock.ACTION_SET_TIMER)
                .putExtra(AlarmClock.EXTRA_MESSAGE, message)
                .putExtra(AlarmClock.EXTRA_LENGTH, seconds)
                .putExtra(AlarmClock.EXTRA_SKIP_UI, true);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivity(intent);
        }
    }

需要注意的是：无论是闹钟还是秒表，都需要在manifest中添加SET_ALARM权限。

    <uses-permission android:name="com.android.alarm.permission.SET_ALARM" />



<a name="anchor2"></a>
# 2. 行程

点击查看：[行程的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/02_event)

添加行程的Intent示意如下：

    public void addEvent(String title, String location, Calendar begin, Calendar end) {
        Intent intent = new Intent(Intent.ACTION_INSERT)
                .setData(Events.CONTENT_URI)
                .putExtra(Events.TITLE, title)
                .putExtra(Events.EVENT_LOCATION, location)
                .putExtra(CalendarContract.EXTRA_EVENT_BEGIN_TIME, begin)
                .putExtra(CalendarContract.EXTRA_EVENT_END_TIME, end);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivity(intent);
        }   
    }   




<a name="anchor3"></a>
# 3. 拍照并获取所拍的图片

点击查看：[获取拍照图片的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/03_camera)


## 3.1 打开Camera进行拍照

    public void capturePhoto() {

        Intent intent = new Intent(MediaStore.ACTION_IMAGE_CAPTURE);  
    
        ContentValues values = new ContentValues();
        values.put(MediaStore.Images.Media.DISPLAY_NAME, "MyImage");  
        values.put(MediaStore.Images.Media.DESCRIPTION, "this is description");  
        values.put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg");  
        mLocationForPhotos = getContentResolver().insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values);  
        intent.putExtra(MediaStore.EXTRA_OUTPUT, mLocationForPhotos);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivityForResult(intent, REQUEST_IMAGE_CAPTURE);
        }   
    }   

## 3.2 获取拍照的图片

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {

        if (requestCode == REQUEST_IMAGE_CAPTURE && resultCode == RESULT_OK) {
            try {
                // 首先取得屏幕对象  
                Display display = getWindowManager().getDefaultDisplay();
                // 获取屏幕的宽和高  
                int dw = display.getWidth();
                int dh = display.getHeight();
                // 获取图片原始大小
                BitmapFactory.Options op = new BitmapFactory.Options();
                op.inJustDecodeBounds = true;
                Bitmap pic = BitmapFactory.decodeStream(
                        this.getContentResolver().openInputStream(mLocationForPhotos),
                        null, op);

                int wRatio = (int) Math.ceil(op.outWidth / (float) dw); //计算宽度比例  
                int hRatio = (int) Math.ceil(op.outHeight / (float) dh); //计算高度比例  
                Log.d(TAG, "wRatio="+wRatio+", hRatio="+hRatio);
                // 设置缩放比例
                if (wRatio > 1 && hRatio > 1) {
                    if (wRatio > hRatio) {
                        op.inSampleSize = wRatio;
                    } else {
                        op.inSampleSize = hRatio;
                    }
                } else {
                    op.inSampleSize = 4; // 默认缩小为4倍 
                }

                op.inJustDecodeBounds = false; //注意这里，一定要设置为false，因为上面我们将其设置为true来获取图片尺寸了  
                pic = BitmapFactory.decodeStream(this.getContentResolver().openInputStream(mLocationForPhotos),
                        null, op);
                mImage.setImageBitmap(pic);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }



<a name="anchor4"></a>
# 4. 获取联系人

点击查看：[获取联系人的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/04_contact)

下面给出获取联系人号码的示例：

    public void selectContact() {
        // Start an activity for the user to pick a phone number from contacts
        Intent intent = new Intent(Intent.ACTION_PICK);
        intent.setType(CommonDataKinds.Phone.CONTENT_TYPE);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivityForResult(intent, REQUEST_SELECT_PHONE_NUMBER);
        }   
    }   

说明：上面函数的作用是选取一个联系人。选取联系人之后，再提取出联系人号码。

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        if (requestCode == REQUEST_SELECT_PHONE_NUMBER && resultCode == RESULT_OK) {
            // Get the URI and query the content provider for the phone number
            Uri contactUri = data.getData();
            String[] projection = new String[]{CommonDataKinds.Phone.NUMBER};
            Cursor cursor = getContentResolver().query(contactUri, projection,
                    null, null, null);
            // If the cursor returned is valid, get the phone number
            if (cursor != null && cursor.moveToFirst()) {
                int numberIndex = cursor.getColumnIndex(CommonDataKinds.Phone.NUMBER);
                String number = cursor.getString(numberIndex);
                Log.d(TAG, "number="+number);
            }   
        }   
    }   




<a name="anchor5"></a>
# 5. Email

点击查看：[Email的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/05_email)

打开Email程序的示意非常简单：

    public void composeEmail(String[] addresses, String subject, Uri attachment) {
        Intent intent = new Intent(Intent.ACTION_SEND);
        intent.setType("*/*");
        intent.putExtra(Intent.EXTRA_EMAIL, addresses);
        intent.putExtra(Intent.EXTRA_SUBJECT, subject);
        intent.putExtra(Intent.EXTRA_STREAM, attachment);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivity(intent);
        }   
    }   






<a name="anchor6"></a>
# 6. 从图库中选择图片

点击查看：[从图库中选择图片的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/06_gallery)

## 6.1 选择图片

    public void selectImage() {
        //Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
        Intent intent = new Intent(Intent.ACTION_OPEN_DOCUMENT);
        intent.setType("image/*");
        intent.addCategory(Intent.CATEGORY_OPENABLE);

        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivityForResult(intent, REQUEST_IMAGE_GET);
        }   
    }   


## 6.2 读取所选择的图片

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {

        if (requestCode == REQUEST_IMAGE_GET && resultCode == RESULT_OK) {

            try {   
                Uri mLocationForPhotos = data.getData();
                // 首先取得屏幕对象  
                Display display = getWindowManager().getDefaultDisplay();  
                // 获取屏幕的宽和高  
                int dw = display.getWidth();  
                int dh = display.getHeight();  
                // 获取图片原始大小
                BitmapFactory.Options op = new BitmapFactory.Options();  
                op.inJustDecodeBounds = true;  
                Bitmap pic = BitmapFactory.decodeStream(
                        this.getContentResolver().openInputStream(mLocationForPhotos),
                        null, op);

                int wRatio = (int) Math.ceil(op.outWidth / (float) dw); //计算宽度比例  
                int hRatio = (int) Math.ceil(op.outHeight / (float) dh); //计算高度比例  
                Log.d(TAG, "wRatio="+wRatio+", hRatio="+hRatio);
                // 设置缩放比例
                if (wRatio > 1 && hRatio > 1) {
                    if (wRatio > hRatio) {
                        op.inSampleSize = wRatio;
                    } else {
                        op.inSampleSize = hRatio;
                    }
                } else {
                    op.inSampleSize = 4; // 默认缩小为4倍 
                }

                op.inJustDecodeBounds = false; //注意这里，一定要设置为false，因为上面我们将其设置为true来获取图片尺寸了  
                pic = BitmapFactory.decodeStream(this.getContentResolver().openInputStream(mLocationForPhotos),
                        null, op);
                mImage.setImageBitmap(pic);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }



<a name="anchor7"></a>
# 7. 打开网页

点击查看：[网页的测试源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/intent/common_intents/07_webview)


    public void openWebPage(String url) {
        Uri webpage = Uri.parse(url);
        Intent intent = new Intent(Intent.ACTION_VIEW, webpage);
        if (intent.resolveActivity(getPackageManager()) != null) {
            startActivity(intent);
        }   
    }   



[link_intent_introduce]: /2014/05/31/Intent/
