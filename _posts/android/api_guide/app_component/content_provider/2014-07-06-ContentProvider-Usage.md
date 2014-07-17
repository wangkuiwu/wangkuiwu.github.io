---
layout: post
title: "Android 之ContentProvider(二)之 ContentProvider基本用法"
description: "android training"
category: android
tags: [android]
date: 2014-07-06 10:11
---

> 本章给出ContentProvider的完整示例，并对之进行介绍。

> **目录**  
> **1**. [ContentProvider简介](#anchor1)  
> **2**. [ContentProvider示例](#anchor2)  


<a name="anchor1"></a>
# ContentProvider简介

ContentProvider通常用于共享数据。

当其他程序需要访问本程序的数据，并且数据的结构比较复杂时，就可以使用ContentProvider来共享数据。如果数据不需要跨程序访问，使用数据库即可；如果数据结构比较简单，可以考虑前面提到的通过Intent共享文本等简单数据，或者通过FileProvider共享文件。


<a name="anchor2"></a>
# ContentProvider示例

接下来，实现一个ContentProvider。该ContentProvider包括两部分：ContentProvider提供者APK 和 ContentProvider测试APK。  
(01) ContentProvider提供者：自定义一个ContentProvider，并监听相应的URI。客户可以通过URI插入/删除/更新/查询数据。ContentProvider中的数据记录的是人的信息，包括"姓名，出生年月，email，性别"等信息。  
(02) ContentProvider测试APK：通过URI向ContentProvider发起插入/删除/更新/查询等操作。  

点击查看：[ContentProvider示例的完整源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/app_components/contentprovider/01_basic/MyProvider)

下面介绍ContentProvider的实现步骤。


## 1. ContentProvider提供者APK

### 1.1 ContentProvider的存储表格

根据ContentProvider的数据特性，我们建立一张表，表格包括"id/姓名/出生年月/email/性别"这些信息。表对应的类如下：


    public final class MyContract {
        public MyContract() {}

        /** 
         * BaseColumns类中有两个属性：_ID 和 _COUNT
         */
        public static abstract class Entry implements BaseColumns {
            public static final String TABLE_NAME = "mytable01";
            public static final String NAME       = "name";
            public static final String BIRTH_DAY  = "birthday";
            public static final String EMAIL      = "email";
            public static final String GENDER     = "gender";
        }   
    }

说明：BaseColumns是Android自带的类，它集成了"_ID"和"_COUNT"两个属性。 


### 1.2 ContentProvider对应的manifest

在manifest中声明我们自定义的ContentProvider。


    <application android:label="@string/app_name" android:icon="@drawable/ic_launcher">
        <provider android:name="MyProvider" android:authorities="com.skw.myprovider" />
    </application>


### 1.3 自定义的ContentProvider类

完成ContentProvider类，主要需要注意以下几点：  
(01) ContentProvider的数据一般是以"数据库"或"网络数据"的方式存储的。如果是数据库，则需要实现SQLiteOpenHelper类。通过SQLiteOpenHelper类新建/管理数据库。  
(02) ContentProvider主要是以Uri的形式方式访问的(也可以通过Intent)。要通过UriMatcher注册ContentProvider监听的Uri。  
(03) ContentProvider是一个抽象类。当我们需要以继承ContentProvider的方式自定义ContentProvider时，需要实现query(), insert(), update(), delete(), getType(), onCreate()这六个函数。  

### 1.3.1 数据库

下面，先介绍ContentProvider的数据库的实现。

    // 创建表格的SQL语句
    private static final String SQL_CREATE_ENTRIES =
        "CREATE TABLE " + Entry.TABLE_NAME + " (" +
        Entry._ID + " INTEGER PRIMARY KEY," +
        Entry.NAME + " TEXT NOT NULL, " +
        Entry.BIRTH_DAY + " TEXT, " +
        Entry.EMAIL + " TEXT, " +
        Entry.GENDER + " INTEGER " +
        " )";

    ...

    private class DBLiteHelper extends SQLiteOpenHelper {
        public static final int DATABASE_VERSION = 1;
        public static final String DATABASE_NAME = "MyProvider.db";

        public DBLiteHelper(Context context) {
            super(context, DATABASE_NAME, null, DATABASE_VERSION);
        }

        @Override
        public void onCreate(SQLiteDatabase db) {
            db.execSQL(SQL_CREATE_ENTRIES);
        }

        @Override
        public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        }
    }

说明：关于SQLiteOpenHelper的内容在"[数据存储章节](/2014/05/30/SavingData/)"中已经详细介绍过了。


### 1.3.2 注册Uri


    // Uri的authority
    public static final String AUTHORITY = "com.skw.myprovider";
    // Uri的path
    public static final String PATH = "table01";
    // UriMatcher中URI对应的序号
    public static final int ITEM_ALL = 1;
    public static final int ITEM_ID  = 2;

    private static final UriMatcher URI_MATCHER = new UriMatcher(UriMatcher.NO_MATCH);
    static {
        URI_MATCHER.addURI(AUTHORITY, PATH, ITEM_ALL);
        URI_MATCHER.addURI(AUTHORITY, PATH+"/#", ITEM_ID);
    }

说明：通过addURI()就可以将URI注册到UriMatcher中，从而实现ContentProvider对URI的监听。这里的AUTHORITY与manifest中的android:authorities一致！  
例如， URI_MATCHER.addURI(AUTHORITY, PATH, ITEM_ALL); 意味着ContentProvider对"content://con.skw.myprovider/table01"进行监听。   
例如， URI_MATCHER.addURI(AUTHORITY, PATH+"/#", ITEM_ALL); 意味着ContentProvider对"content://con.skw.myprovider/table01/5"进行监听。   


### 1.3.2 实现ContentProvider的抽象函数

#### onCreate()

    @Override
    public boolean onCreate() {
        mDbHelper = new DBLiteHelper(this.getContext());
        Log.d(TAG, "open/create table");
        return true;
    }

说明：onCreate()中新建SQLiteOpenHelper对象。


#### delete()

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        SQLiteDatabase db = mDbHelper.getWritableDatabase();

        int count = 0;
        switch (URI_MATCHER.match(uri)) {
        case ITEM_ALL:
            count = db.delete(Entry.TABLE_NAME, selection, selectionArgs);
            Log.d(TAG, "delete ITEM uri="+uri+", count="+count);
            break;
        case ITEM_ID:
            // 获取id列的值
            String id = uri.getPathSegments().get(1);
            count = db.delete(Entry.TABLE_NAME, Entry._ID+"=?", new String[]{id});
            Log.d(TAG, "delete ITEM_ID id="+id+", uri="+uri+", count="+count);
            break;
        default:
            throw new IllegalArgumentException("Unknown URI"+uri);
        }
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }

说明：delete()中会通过match()获取uri对应的编码。这里的编码就是和addURI()注册uri的编码是相对应的。此外，notifyChange()的作用是通常数据库变化，若有ContentObserver监听该Uri，则notifyChange()最终会将消息传递给监听者。

insert(), update(), query()的实现与delete()类似，就不再说明。

#### getType()

    @Override
    public String getType(Uri uri) {
        switch (URI_MATCHER.match(uri)) {
        case ITEM_ALL:
            return "skw.myprovider.dir/table01";
        case ITEM_ID:
            return "skw.myprovider.item/table01";
        default:
            throw new IllegalArgumentException("Unknown URI"+uri);
        }
    }

说明：getType()是返回Uri对应的数据类型。



## 2. ContentProvider测试APK


    public class ProviderTest extends Activity 
        implements View.OnClickListener {

        private static final String TAG = "##ProviderTest##";

        // 数据库的属性，与MyProvider的表格属性一致
        public static final String NAME      = "name";
        public static final String BIRTH_DAY = "birthday";
        public static final String EMAIL     = "email";
        public static final String GENDER    = "gender";
        // 数据库的URI
        public static final Uri CONTENT_URI = Uri.parse("content://com.skw.myprovider/table01");

        private ContentResolver mContentResolver = null;
        /** Called when the activity is first created. */

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            ((Button)findViewById(R.id.insert)).setOnClickListener(this);
            ((Button)findViewById(R.id.deleteFirst)).setOnClickListener(this);
            ((Button)findViewById(R.id.deleteKate)).setOnClickListener(this);
            ((Button)findViewById(R.id.deleteAll)).setOnClickListener(this);
            ((Button)findViewById(R.id.update)).setOnClickListener(this);
            ((Button)findViewById(R.id.show)).setOnClickListener(this);

            // 删除第一行，然后全部打印出来
            mContentResolver = getContentResolver();

        }


        @Override
        public void onClick(View view) {
            switch(view.getId()) {
                case R.id.insert:
                    // 添加
                    insert("Jimmy", "20020201", "Jimmy20020201@126.com", 1);
                    insert("Kate",  "20030104", "kate20030104@126.com", 0);
                    insert("Li Lei", "20021124", "lilei20101124@126.com", 1);
                    insert("Lucy", "20010624", "lucy20101124@126.com", 0);
                    break;
                case R.id.deleteFirst:
                    ContentUris cus = new ContentUris();
                    Uri uri = cus.withAppendedId(CONTENT_URI, 1);
                    Log.d(TAG, "delete uri="+uri);
                    mContentResolver.delete(uri, null, null);
                    break;
                case R.id.deleteKate:
                    // 删除“username=Kate”的行，然后全部打印出来
                    mContentResolver.delete(CONTENT_URI, NAME+"=?", new String[]{"Kate"});
                    break;
                case R.id.deleteAll:
                    // 删除全部的行，然后全部打印出来
                    deleteAll() ;
                    break;
                case R.id.update:
                    // 更新第1个值，然后全部打印出来
                    updateItem() ;
                    break;
                case R.id.show:
                    // 打印全部的值
                    printAll() ;
                    break;
                default:
                    // 查找第2个值
                    //querySecondItem() ;
                    break;
            }
        }

        /*
         * 通过ContentResolver,将值插入到MyProvider中
         */
        private void insert(String name, String date, String email, int gender) {

            ContentResolver cr = getContentResolver();

            ContentValues cv = new ContentValues();
            cv.put(NAME, name);
            cv.put(BIRTH_DAY, date);
            cv.put(EMAIL, email);
            cv.put(GENDER, gender);
            Uri uri = cr.insert(CONTENT_URI, cv);
            Log.d(TAG, "insert uri="+uri);
        }

        private void updateItem() {
            ContentResolver cr = getContentResolver();

            ContentUris cus = new ContentUris();
            Uri uri = cus.withAppendedId(CONTENT_URI, 1);

            ContentValues cv = new ContentValues();
            cv.put(NAME, "update_name");
            cv.put(BIRTH_DAY, "update_date");
            cv.put(EMAIL, "update_email");
            cv.put(GENDER, 1);
            cr.update(uri, cv, null, null);
        }

        /*
         * 通过ContentResolver,将MyProvider中的值全部删除
         */
        private void deleteAll() {

            Log.d(TAG, "delete all value!");
            ContentResolver cr = getContentResolver();
            cr.delete(CONTENT_URI, null, null);
        }

        private void querySecondItem() {
            ContentResolver cr = getContentResolver();
            ContentUris cus = new ContentUris();
            Uri uri = cus.withAppendedId(CONTENT_URI, 2);
            String[] proj = new String[] { NAME, BIRTH_DAY, EMAIL, GENDER};
            Cursor cursor = cr.query(uri, proj, null, null, null);
            int index = 0;
            while (cursor.moveToNext()) {
                Log.d(TAG, "querySecondItem--"+index+"--"
                        +", email=" + cursor.getString(cursor.getColumnIndex(EMAIL))
                        +", username=" + cursor.getString(cursor.getColumnIndex(NAME))
                        +", date=" + cursor.getString(cursor.getColumnIndex(BIRTH_DAY))
                        +", gender=" + cursor.getInt(cursor.getColumnIndex(GENDER)));
                index++;
            }
        }

        private void printAll() {
            //通过contentResolver进行查找
            ContentResolver cr = getContentResolver();

            Log.d(TAG, "print all value!");
            // query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)
            // 返回的列
            String[] proj = new String[] { NAME, BIRTH_DAY, EMAIL, GENDER};
            Cursor cursor = cr.query(
                CONTENT_URI, proj, null, null, null);
            int index = 0;
            while (cursor.moveToNext()) {
                Log.d(TAG, "printAll--"+index+"--"
                        +", email=" + cursor.getString(cursor.getColumnIndex(EMAIL))
                        +", username=" + cursor.getString(cursor.getColumnIndex(NAME))
                        +", date=" + cursor.getString(cursor.getColumnIndex(BIRTH_DAY))
                        +", gender=" + cursor.getString(cursor.getColumnIndex(GENDER)));
                index++;
            }
            startManagingCursor(cursor);  //查找后关闭游标
        }
    }


