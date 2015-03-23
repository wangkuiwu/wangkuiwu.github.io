---
layout: post
title: "Android培训(二)共享篇02之 共享文件"
description: "android training"
category: android
tags: [android]
date: 2014-01-12 09:25
---

> 本章介绍如何通过FileProvider来共享文件。该文件是对应"[FileProvider测试源码](https://github.com/wangkuiwu/android_applets/tree/master/training/02_sharing_data/02_share_file/ShareFile)"进行的说明

> **目录**  
[第1部分 设置文件共享的配置](#anchor1)  
[第2部分 提供文件共享的选择界面](#anchor2)  
[第3部分 发送请求并处理返回结果](#anchor3)  


<a name="anchor1"></a>
# 第1部分 设置文件共享的配置

Android提供了FileProvider来提供不同程序之间的文件共享。下面就说说如何通过FileProvider来实现不同程序之间的文件共享。

## 1. 在manifest中声明FileProvider

    <provider
        android:name="android.support.v4.content.FileProvider"
        android:authorities="com.skw.sharefile.fileprovider"
        android:grantUriPermissions="true"
        android:exported="false">
        <meta-data
            android:name="android.support.FILE_PROVIDER_PATHS"
            android:resource="@xml/filepaths" />
    </provider>

说明：

(01) android:name -- 指定该provider是FileProvider类型。FileProvider是android-support-v4.jar包提供的功能。  
(02) android.authorities -- 指定FileProvider的URI authority。定义该名称是，请尽量使用自己的"包名+.fileprovider"。  
(03) android.resource -- 指定被共享文件的配置文件路径。在res/xml/filepaths.xml中配置了共享文件的路径。  



## 2. 定义共享文件对应的配置文件

res/xml/filepaths.xml的内容如下：

    <paths>
        <files-path path="images/" name="myimages" />
    </paths>

说明：filepaths.xml的作用是指定该程序所共享文件所在的目录。path="images/"，它是告诉被共享的文件是在该程序的files/images/目录下(每个apk都有自己的files文件夹)；name="myimages"，它是告诉FileProvider的URI的"path segment"的名称是"myimages"。

假如要共享的文件是default_image.jpg。那么，经过上面的定义之后，对应的URI如下：

    content://com.example.myapp.fileprovider/myimages/default_image.jpg




<a name="anchor2"></a>
# 第2部分 提供文件共享的选择界面

经过上面的设置之后，程序就能提供文件共享功能。通常，会对应的提供文件共享的选择界面给用户，方便他们做出选择。下面介绍选择界面的实现方法。


## 1. 定义选择界面对应的Activity


        <activity android:name=".FileSelector" >
            <intent-filter>    
                <action android:name="android.intent.action.PICK" />
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.OPENABLE" />
                <data android:mimeType="image/*" />
                <data android:mimeType="text/plain" />
            </intent-filter>   
        </activity> 


说明：

(01) 系统默认的选择图片动作是ACTION_PICK(即，android.intent.action.PICK)。  
(02) FileSelector接受的文件类型包括"图片"和"文本"。


## 2. 显示图片列表

在FileSelector中通过ListView显示图片。

    @Override
    public void onCreate(Bundle savedInstanceState) {

        // the "images" is correspond to the "path" property of res/xml/filepaths
        mImagesDir = new File(getFilesDir(), "images");
        if (!mImagesDir.exists())
            mImagesDir.mkdir();
        // get image under .../file/images/
        mImageFiles = mImagesDir.listFiles();

        ...

        ListView mListView = (ListView)findViewById(R.id.lv_choose);
        MyAdapter adapter = new MyAdapter(this);
        mListView.setAdapter(adapter);
        mListView.setOnItemClickListener(this);

        ...
    }


### 3. 响应文件选择

在用户选择界面，当用户选择某一图片之后，我们通过FileProvider并提供相应的权限的方式将文件共享出去。

    @Override
    public void onItemClick(AdapterView<?> adapterView, View view, int position, long rowId) {
        File requestFile = mImageFiles[position];

        Uri fileUri=null;
        try {
            fileUri = FileProvider.getUriForFile(FileSelector.this, "com.skw.sharefile.fileprovider", requestFile);
        } catch (IllegalArgumentException e) {
            e.printStackTrace();
        }

        if(fileUri != null) {
            Intent resultIntent = new Intent();
            resultIntent.setFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
            resultIntent.setDataAndType(fileUri, getContentResolver().getType(fileUri));
            FileSelector.this.setResult(Activity.RESULT_OK, resultIntent);
            finish();
        }
    }

说明：

(01) 首先，获取到File对象。然后，将该对象传入getUriForFile，并传入在manifest中定义的<android:authorities>。目的是获取到Uri对象。  
(02) 将Uri对象作为Intent数据，返回给请求Activity。并设置可读权限 -- Intent.FLAG_GRANT_READ_URI_PERMISSION。

注意：通过setFlags()添加权限是唯一安全的方式，它授予的权限是临时的。该权限在接受应用的任务栈被完成之后，会自动销毁。这样，就避免对文件URI调用Context.grantUriPermission()，因为通过该方法授予的权限，你只能通过调用Context.revokeUriPermission()来撤销。




<a name="anchor3"></a>
# 第3部分 发送请求并处理返回结果

前面介绍了如何共享文件，提供共享文件选择界面，以及将选择结果返回给客户。这里，将介绍如何请求获取共享文件，以及收到请求结果之后的处理方法。


## 1. 发送请求

    @Override
    public void onCreate(Bundle savedInstanceState) {
        // 请求按钮
        Button mSelect = (Button) findViewById(R.id.bt_select);
        mSelect.setOnClickListener(this);

        // 请求Intent
        mRequestFileIntent = new Intent(Intent.ACTION_PICK);
        mRequestFileIntent.setType("image/jpg");
    }   

    @Override 
    public void onClick(View v) {
        // 发送请求
        startActivityForResult(mRequestFileIntent, 0); 
    } 

说明：请求Intent中指出了请求的Intent动作，以及对应的类型。


## 2. 处理请求返回结果

    @Override
    public void onActivityResult(int requestCode, int resultCode,
            Intent intent) {
        if (resultCode != RESULT_OK) {
            return ;
        }   
            
        Uri uri = intent.getData();
        try {
            ParcelFileDescriptor mInputPFD = getContentResolver().openFileDescriptor(uri, "r");
            // 获取请求图片
            if (mInputPFD != null) {
                FileDescriptor fd = mInputPFD.getFileDescriptor();
                Bitmap bitmap = BitmapFactory.decodeFileDescriptor(fd);
                mHead.setImageBitmap(bitmap);
                Log.d(TAG, "get bitmap success!");
            }   
        } catch (FileNotFoundException e) {
            e.printStackTrace();
            Log.e(TAG, "File not found.");
            return;
        }

        // 获取请求文件的"名称"和"大小"
        Cursor returnCursor = getContentResolver().query(uri, null, null, null, null);
        int nameIndex = returnCursor.getColumnIndex(OpenableColumns.DISPLAY_NAME);
        int sizeIndex = returnCursor.getColumnIndex(OpenableColumns.SIZE);
        returnCursor.moveToFirst();

        String name = returnCursor.getString(nameIndex);
        long size = returnCursor.getLong(sizeIndex);
    }

说明：

(01) 从返回结果中通过openFileDescriptor()获取ParcelFileDescriptor对象，然后再通过ParcelFileDescriptor对象获取图片。  
(02) FileProvider类有一个默认的query()方法的实现，它返回一个Cursor，它包含了URI所关联的文件的名字和尺寸。 

