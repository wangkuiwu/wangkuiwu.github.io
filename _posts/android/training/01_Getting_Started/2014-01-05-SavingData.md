---
layout: post
title: "Android培训(一)开始篇05之 保存数据"
description: "android training"
category: android
tags: [android]
date: 2014-01-05 19:25
---

> 本章介绍Android中常用的保存数据的几种基本方法。

> **目录**  
[第1部分 SharedPreferences保存数据](#anchor1)  
[第2部分 File保存数据](#anchor2)  
[第3部分 SQLite保存数据](#anchor3)  


<a name="anchor1"></a>
# 第1部分 SharedPreferences保存数据

## 1. 基本用法

### 获取SharedPreferences对象

在Activity中有两种获取SharedPreferences的方法：   
(01) getSharedPreferences(name, mode) -- 该方法能够指定SharedPreferences的名字。  
(02) getPreferences(mode) -- 该方法会采用该Activity的名称作为名字。  
注意：不同Activity调用getPreferences()获取到的SharedPreferences并不相同！

        // 获取SharedPreferences方法一：指定名称
        mPref = getSharedPreferences(SP_FILE_NAME, MODE_PRIVATE); 
        // 获取SharedPreferences方法二：apk默认
        mPref = getPreferences(MODE_PRIVATE);


### 写数据

首先获取SharedPreferences.Editor对象，通过该对象才能写入数据。

    mEditor = mPref.edit();

接着，就可以通过mEditor来将数据保存到SharedPreferences中。SharedPreferences中的数据是以key-value键值对的形式存在的。另外，添加了数据之后，记得调用commit()接口提交数据。

    private void writePref() {
        mEditor.putString(SP_NAME, "jim");      // 名字
        mEditor.putInt(SP_AGE, 25);             // 年龄
        mEditor.putBoolean(SP_SINGLE, false);   // 婚否

        // 提交数据
        mEditor.commit();
    }   

### 读数据

直接调用SharedPreferences的相应接口就能读取数据。

    private void readPref() {
        String name = mPref.getString(SP_NAME, null); 
        int age = mPref.getInt(SP_AGE, 1); 
        boolean single = mPref.getBoolean(SP_SINGLE, false); 
    }   

点击查看：[SharedPreferences示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/06_saving_data/01_shared_preferences/01_basic)





<a name="anchor2"></a>
# 第2部分 File保存数据

将数据存储成文件，并保存到设备上。可以选择Exernal(外部) 和 Internal(内部)存储设别。

## 1. 打开文件

可以通过Android自带的openFileOutput()接口，也可以使用Java的接口。

    // Android自带的openFileOutput()接口
    FileOutputStream outputStream = openFileOutput(filename, Context.MODE_PRIVATE);

    // Java的文件接口,file是File对象
    File file = new File(getFilesDir(), filename);
    FileOutputStream outputStream = new FileOutputStream(file);

说明：使用Java的文件接口创建FileOutputStream时，需要指定File对象。创建File对象时，可以通过getFilesDir()来获取当前程序的文件路径，该路径是Android为每个程序提供的默认的保存文件的路径。此外，Android还提供了getCacheDir()来获取当前程序的缓存区路径。


## 2. 删除文件

可以通过Android自带的deleteFile()接口，也可以使用Java的接口。

    // Android自带的deleteFile()接口
    deleteFile(filename);

    // Java的文件接口,file是File对象
    file.delete();

## 3. 创建临时文件

若要创建临时文件，可以直接使用File的createTempFile()接口。示例如下：

    File file = File.createTempFile(filename, null, getCacheDir());


点击查看：[File示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/06_saving_data/02_files/FileTest)



<a name="anchor3"></a>
# 第3部分 SQLite保存数据

## 1. 数据库设计建议

**建议一**：Android官方建立我们在使用数据库时，创建一个与数据库对应的Contract类。Contract类如何定义，以及作用是什么呢？

在Contract中，将我们需要用到的数据库相关的"URI，数据库表格、表格属性"等内容定义在其中。这样做的目的是，一是结构更加清晰；再者，在将一些URI、表格属性作为public static类型成员之后，可以方便我们随时随地调用。


**建议二**：定义表格的时候，实现BaseColumns接口。因为BaseColumns中定义了_ID属性。这样做的目的，是让你的表格更加通用。


**示例**

    public final class FeedReaderContract {
        // To prevent someone from accidentally instantiating the contract class,
        // give it an empty constructor.
        public FeedReaderContract() {}

        /* Inner class that defines the table contents */
        public static abstract class FeedEntry implements BaseColumns {
            public static final String TABLE_NAME = "entry";
            public static final String COLUMN_NAME_ENTRY_ID = "entryid";
            public static final String COLUMN_NAME_TITLE = "title";
            public static final String COLUMN_NAME_CONTENT = "content";
            public static final String COLUMN_NAME_SUBTITLE = "subtitle";
            public static final String COLUMN_NAME_NULLABLE = ""; 
            //...
        }   
    }

说明：上面是FeedReader数据库对应的Contract类。在FeedReaderContract中定义了一个表格对应的FeedEntry，它实现了BaseColumns接口。


## 2. SQLiteOpenHelper

SQLiteOpenHelper是一个抽象类。它的作用是用来对数据库的创建和版本进行管理！

当我们自定义SQliteOpenHelper的子类时，必须要实现onCreate()和onUpgrade()函数。   
onCreate(): 数据库第一次被创建时执行。数据库的创建方法，是调用SQLiteOpenHelper的getReadableDatabase()或getWritableDatabase()接口。  
onUpgrade(): 数据库需要被更新时时执行。  

    private static final String TEXT_TYPE = " TEXT";
    private static final String COMMA_SEP = ",";
    private static final String SQL_CREATE_ENTRIES =
        "CREATE TABLE " + FeedEntry.TABLE_NAME + " (" +
        FeedEntry._ID + " INTEGER PRIMARY KEY," +
        FeedEntry.COLUMN_NAME_ENTRY_ID + TEXT_TYPE + COMMA_SEP +
        FeedEntry.COLUMN_NAME_TITLE + TEXT_TYPE + COMMA_SEP +
        FeedEntry.COLUMN_NAME_CONTENT + TEXT_TYPE +
        " )";

    private static final String SQL_DELETE_ENTRIES =
        "DROP TABLE IF EXISTS " + FeedEntry.TABLE_NAME;
    
    ...
    
    public class FeedReaderDbHelper extends SQLiteOpenHelper {
        // If you change the database schema, you must increment the database version.
        public static final int DATABASE_VERSION = 1;
        public static final String DATABASE_NAME = "FeedReader.db";

        public FeedReaderDbHelper(Context context) {
            super(context, DATABASE_NAME, null, DATABASE_VERSION);
            Log.d(TAG, "FeedReaderDbHelper");
        }

        public void onCreate(SQLiteDatabase db) {
            Log.d(TAG, "FeedReaderDbHelper: onCreate");
            db.execSQL(SQL_CREATE_ENTRIES);
        }
        public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
            Log.d(TAG, "FeedReaderDbHelper: onUpgrade");
            // This database is only a cache for online data, so its upgrade policy is
            // to simply to discard the data and start over
            db.execSQL(SQL_DELETE_ENTRIES);
            onCreate(db);
        }
        public void onDowngrade(SQLiteDatabase db, int oldVersion, int newVersion) {
            onUpgrade(db, oldVersion, newVersion);
        }
    }

说明：FeedReaderDbHelper的onCreate()只有在数据库第一次被创建的时候才调用。在创建数据库时，我们创建了表格。


## 3. 数据库操作

### 3.1 获取SQLiteOpenHelper对象

onCreate(): 数据库第一次被创建时执行。数据库的创建方法，是调用SQLiteOpenHelper的getReadableDatabase()或getWritableDatabase()接口。  

    // 创建SQLiteOpenHelper对象
    FeedReaderDbHelper mDbHelper = new FeedReaderDbHelper(getApplicationContext());
    
    // 以读写方式打开数据库
    SQLiteDatabase db = mDbHelper.getWritableDatabase();
    // 以只读方式打开数据库
    SQLiteDatabase db = mDbHelper.getReadableDatabase();


### 3.2 插入数据

在将数据插入到数据库时，建议使用ContentValues对象。ContentValues的本质是key-value键值对，在用ContentValues操作数据库时，key对应的值要是表格属性，而value则要是属性对应的值。

    private void testWriteDatabase(int id, String title, String content) {
        // Gets the data repository in write mode
        SQLiteDatabase db = mDbHelper.getWritableDatabase();

        // Create a new map of values, where column names are the keys
        ContentValues values = new ContentValues();
        values.put(FeedEntry.COLUMN_NAME_ENTRY_ID, id);
        values.put(FeedEntry.COLUMN_NAME_TITLE, title);
        values.put(FeedEntry.COLUMN_NAME_CONTENT, content);

        // Insert the new row, returning the primary key value of the new row
        long newRowId;
        newRowId = db.insert(
                 FeedEntry.TABLE_NAME,
                 FeedEntry.COLUMN_NAME_NULLABLE,
                 values);
    }


### 3.3 查询和读取数据

在查询和读取数据库中数据时，建议使用ContentValues对象。ContentValues的本质是key-value键值对，在用ContentValues操作数据库时，key对应的值要是表格属性，而value则要是属性对应的值。

    private void testReadDatabase() {
        SQLiteDatabase db = mDbHelper.getReadableDatabase();

        // Define a projection that specifies which columns from the database
        // you will actually use after this query.
        String[] projection = {
            FeedEntry._ID,
            FeedEntry.COLUMN_NAME_ENTRY_ID,
            FeedEntry.COLUMN_NAME_TITLE,
            FeedEntry.COLUMN_NAME_CONTENT,
            };
        String selection = FeedEntry.COLUMN_NAME_ENTRY_ID + "=?";
        String[] selectionArgs = new String[] {"1"};

        Cursor c = db.query(
            FeedEntry.TABLE_NAME,  // 被查询的表格
            projection,            // 返回属性：查询结果表中包含的"属性"
            selection,             // 查找条件(1)：相当于where的前半段
            selectionArgs,         // 查找条件(2)：相当于where的后半段
            null,                  // don't group the rows
            null,                  // don't filter by row groups
            null                   // don't sort 
            );

        Log.d(TAG, "read Database");
        while(c.moveToNext()) {
            // 
            int id = c.getInt(c.getColumnIndex(FeedEntry._ID));
            int entryid = c.getInt(c.getColumnIndex(FeedEntry.COLUMN_NAME_ENTRY_ID));
            String title = c.getString(c.getColumnIndex(FeedEntry.COLUMN_NAME_TITLE));
            String content = c.getString(c.getColumnIndex(FeedEntry.COLUMN_NAME_CONTENT));
            Log.d(TAG, "id="+id+", entryid="+entryid+", title="+title+", content="+content);
        }
    }



### 3.4 更新数据

在更新数据库中数据时，建议使用ContentValues对象。ContentValues的本质是key-value键值对，在用ContentValues操作数据库时，key对应的值要是表格属性，而value则要是属性对应的值。


    private void testUpdateDatabase() {
        SQLiteDatabase db = mDbHelper.getReadableDatabase();

        // New value for one column
        ContentValues values = new ContentValues();
        values.put(FeedEntry.COLUMN_NAME_TITLE, "text updated!");

        // Which row to update, based on the ID
        String selection = FeedEntry.COLUMN_NAME_ENTRY_ID + " LIKE ?";
        String[] selectionArgs = { String.valueOf(1) };

        int count = db.update(
            FeedEntry.TABLE_NAME,
            values,
            selection,
            selectionArgs);
    }




### 3.5 删除数据

在将数据从数据库中删除时，直接使用SQLiteDatabase的delete()接口即可。

    private void testDeleteDatabase() {
        SQLiteDatabase db = mDbHelper.getWritableDatabase();

        // Define 'where' part of query.
        String selection = FeedEntry.COLUMN_NAME_ENTRY_ID + " LIKE ?";
        // Specify arguments in placeholder order.
        String[] selectionArgs = { "1" };
        // Issue SQL statement.
        db.delete(FeedEntry.TABLE_NAME, selection, selectionArgs);
    }


点击查看：[SQLite数据库示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/training/01_getting_started/06_saving_data/03_database/FeedReader)
