---
layout: post
title: "Android网络之HTTP篇(二) HttpClient"
description: "android network"
category: android
tags: [android]
date: 2015-05-02 09:02
---

> 在Android上发送HTTP请求的方式一般有2种：httpClient 和 HttpURLConnection。本文我们就来学习httpClient。

> **目录**  
[1. HttpClient概述](#anchor1)  
[2. HttpClient框架](#anchor2)  
[3. DefaultHttpClient示例](#anchor3)  
[4. AndroidHttpClient示例](#anchor4)  


<a name="anchor1"></a>
# 1. HttpClient概述

HttpClient是Apache开源组织提供的HTTP网络访问接口。  
HttpClient是一个简单的HTTP客户端，可以发送HTTP请求，接受HTTP响应。但是，它不会缓存服务器的响应，不能执行HTTP页面中签入嵌入的JS代码，自然也不会对页面内容进行任何解析、处理；这些工作都需要开发人员来完成的。

HttpClient开源项目的下载地址: [http://hc.apache.org/downloads.cgi](http://hc.apache.org/downloads.cgi)

<br/>
对于Android而言，它已经成功集成了HttpClient。  
HttpClient其实是一个接口类型，它封装了对象需要执行的Http请求、身份验证、连接管理和其它特性。目前，在Android中，HttpClient包含2个实现类：**DefaultHttpClient** 和 **AndroidHttpClient**。


<br/>
## 1.1 DefaultHttpClient和AndroidHttpClient比较

(1) AndroidHttpClient定义在android.net.http.AndroidHttpClient包下，属于Android原生的http访问；而DefaultHttpClient定义在org.apache.http.impl.client.DefaultHttpClient包下，属于对apche项目的支持。  
(2) AndroidHttpClient没有公开的构造函数，只能通过静态方法newInstance()方法来获得AndroidHttpClient对象；而DefaultHttpClient则有public的构造函数。

此外，AndroidHttpClient对于DefaultHttpClient做了一些改进，使其更使用用于Android项目：  
(1) 关掉过期检查，自连接可以打破所有的时间限制。  
(2) 可以设置ConnectionTimeOut（连接超时）和SoTimeout（读取数据超时）。  
(3) 关掉重定向。  
(4) 使用一个Session缓冲用于SSL Sockets。  
(5) 如果服务器支持，使用gzip压缩方式用于在服务端和客户端传递的数据。  
(6) 默认情况下不保留Cookie。


<br/>
## 1.2 使用HttpClient流程

用HttpClient(DefaultHttpClient或AndroidHttpClient)发送请求、接收响应都很简单，只需要几个步骤即可：

第1步：**创建HttpClient对象**。  
第2步：**创建对应的发送请求的对象，并设置对应的参数**。如果需要发送GET请求，则创建HttpGet对象；如果需要发送POST请求，则创建HttpPost对象。  
 &nbsp;&nbsp;&nbsp;&nbsp;  注：若要设置Http请求的参数，GET和POST的设置方式不同，GET方式可以使用拼接字符串的方式，把参数拼接在URL结尾；POST方式需要使用setEntity(HttpEntity entity)方法来设置请求参数。  
第3步：**调用HttpClient对象的execute(HttpUriRequest request)发送请求；执行该方法后返回一个HttpResponse对象**。  
 &nbsp;&nbsp;&nbsp;&nbsp;  注：HttpResponse包含了服务器返回给我们的响应数据，调用HttpResponse的对应方法就能获取服务器的响应头、响应内容等。  
第4步：**检查响应状态是否正常**。HttpResponse中包含一个响应码：若响应码为200，正常；响应码为404，客户端错误；响应码为505，服务器端错误。  
第5步：**获得相应对象当中的数据**。



<a name="anchor2"></a>
# 2. HttpClient框架

httpClient框架图如下：[TODO]


<a name="anchor3"></a>
# 3. DefaultHttpClient示例

示例说明：点击按钮，通过DefaultHttpClient发送HttpGet请求给百度(http://www.baidu.com)，并显示返回结果。

点击下载：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/networks/http/HttpClient/get)

layout文件(demo1.xml)

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >

        <Button
            android:id="@+id/button1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Send Request" />

        <ScrollView
            android:layout_width="match_parent"
            android:layout_height="match_parent" >

            <TextView
                android:id="@+id/TextView1"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Empty" />
        </ScrollView>
    </LinearLayout>


代码(Demo1.java)

    package com.skw.test;

    import org.apache.http.HttpEntity;
    import org.apache.http.HttpResponse;
    import org.apache.http.client.HttpClient;
    import org.apache.http.client.methods.HttpGet;
    import org.apache.http.impl.client.DefaultHttpClient;
    import org.apache.http.util.EntityUtils;
    import android.app.Activity;
    import android.os.Bundle;
    import android.os.Handler;
    import android.os.Message;
    import android.view.View;
    import android.view.View.OnClickListener;
    import android.widget.Button;
    import android.widget.TextView;

    /**
     * DefaultHttpClient测试程序
     *
     * @author skywang
     * @e-mail kuiwu-wang@163.com
     */
    public class Demo1 extends Activity {

        public static final int SHOW_RESPONSE = 0;
        
        private Button button_sendRequest;
        private TextView textView_response;
        
        //新建Handler的对象，在这里接收Message，然后更新TextView控件的内容
        private Handler handler = new Handler() {

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                case SHOW_RESPONSE:
                    String response = (String) msg.obj;
                    textView_response.setText(response);
                    break;

                default:
                    break;
                }            
            }

        };
        
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.demo1);
            textView_response = (TextView)findViewById(R.id.TextView1);
            button_sendRequest = (Button)findViewById(R.id.button1);
            
            button_sendRequest.setOnClickListener(new OnClickListener() {
                
                //点击按钮时，执行sendRequestWithHttpClient()方法里面的线程
                @Override
                public void onClick(View v) {
                    // TODO Auto-generated method stub
                    sendRequestWithHttpClient();
                }
            });
        }

        //方法：发送网络请求，获取百度首页的数据。在里面开启线程
        private void sendRequestWithHttpClient() {
            new Thread(new Runnable() {
                
                @Override
                public void run() {
                    //用HttpClient发送请求，分为五步
                    //第一步：创建HttpClient对象
                    HttpClient httpClient = new DefaultHttpClient();
                    //第二步：创建代表请求的对象,参数是访问的服务器地址
                    HttpGet httpGet = new HttpGet("http://www.baidu.com");
                    
                    try {
                        //第三步：执行请求，获取服务器返还的相应对象
                        HttpResponse httpResponse = httpClient.execute(httpGet);
                        //第四步：检查相应的状态是否正常：检查状态码的值是200表示正常
                        if (httpResponse.getStatusLine().getStatusCode() == 200) {
                            //第五步：从相应对象当中取出数据，放到entity当中
                            HttpEntity entity = httpResponse.getEntity();
                            String response = EntityUtils.toString(entity,"utf-8");//将entity当中的数据转换为字符串
                            
                            //在子线程中将Message对象发出去
                            Message message = new Message();
                            message.what = SHOW_RESPONSE;
                            message.obj = response.toString();
                            handler.sendMessage(message);
                        }
                    } catch (Exception e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                }
            }).start();//这个start()方法不要忘记了        
            
        }    
    }

在manifest中添加网络权限

    <uses-permission android:name="android.permission.INTERNET"/>


<a name="anchor4"></a>
# 4. AndroidHttpClient示例

示例说明：点击按钮，通过DefaultHttpClient发送HttpGet请求给百度(http://www.baidu.com)，并显示返回结果。

点击下载：[示例源码](https://github.com/wangkuiwu/android_applets/tree/master/api_guide/networks/http/HttpClient/get)

代码说明：只需要在DefaultHttpClient的例子中将 `HttpClient httpClient = new DefaultHttpClient();` 修改为 `HttpClient httpCient = AndroidHttpClient.newInstance("");` 即可。

具体如下：

layout文件(demo2.xml)

    <?xml version="1.0" encoding="utf-8"?>
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical" >

        <Button
            android:id="@+id/button1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Send Request" />

        <ScrollView
            android:layout_width="match_parent"
            android:layout_height="match_parent" >

            <TextView
                android:id="@+id/TextView1"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:text="Empty" />
        </ScrollView>

    </LinearLayout>

代码(Demo2.java)

    package com.skw.test;

    import android.net.http.AndroidHttpClient;
    import org.apache.http.Header;
    import org.apache.http.HttpEntity;
    import org.apache.http.HttpResponse;
    import org.apache.http.client.HttpClient;
    import org.apache.http.client.methods.HttpGet;
    import org.apache.http.impl.client.DefaultHttpClient;
    import org.apache.http.util.EntityUtils;
    import android.app.Activity;
    import android.os.Bundle;
    import android.os.Handler;
    import android.os.Message;
    import android.util.Log;
    import android.view.View;
    import android.view.View.OnClickListener;
    import android.widget.Button;
    import android.widget.Toast;
    import android.widget.TextView;

    /**
     * AndroidHttpClient测试程序(Get获取数据)
     *
     * @author skywang
     * @e-mail kuiwu-wang@163.com
     */
    public class Demo2 extends Activity {

        public static final int SHOW_RESPONSE = 0;
        
        private Button button_sendRequest;
        private TextView textView_response;
        
        //新建Handler的对象，在这里接收Message，然后更新TextView控件的内容
        private Handler handler = new Handler() {

            @Override
            public void handleMessage(Message msg) {
                super.handleMessage(msg);
                switch (msg.what) {
                case SHOW_RESPONSE:
                    String response = (String) msg.obj;
                    textView_response.setText(response);
                    break;

                default:
                    break;
                }            
            }

        };
        
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.demo2);
            textView_response = (TextView)findViewById(R.id.TextView1);
            button_sendRequest = (Button)findViewById(R.id.button1);
            
            button_sendRequest.setOnClickListener(new OnClickListener() {
                
                //点击按钮时，执行sendRequestWithHttpClient()方法里面的线程
                @Override
                public void onClick(View v) {
                    // TODO Auto-generated method stub
                    sendRequestWithHttpClient();
                }
            });
        }

        //方法：发送网络请求，获取百度首页的数据。在里面开启线程
        private void sendRequestWithHttpClient() {
            new Thread(new Runnable() {
                
                @Override
                public void run() {
                    //用HttpClient发送请求，分为五步
                    // Toast.makeText(Demo2.this, "AndroidHttpClient", Toast.LENGTH_SHORT).show();
                    Log.d("http01", "AndroidHttpClient");
                    HttpClient httpCient = AndroidHttpClient.newInstance("");
                    HttpGet httpGet = new HttpGet("http://www.baidu.com");
                    
                    // 放入请求头的内容，必须是以键值对的形式，这里以Accept-language为例
                    httpGet.addHeader("Accept-Language","zh-CN,zh;q=0.8,en;q=0.6,zh-TW;q=0.4");
                    // 获取请求头，并用Header数组接收
                    Header [] reqHeaders = httpGet.getAllHeaders();
                    //遍历Header数组，并打印出来
                    for (int i = 0; i < reqHeaders.length; i++) {
                        String name = reqHeaders[i].getName();
                        String value = reqHeaders[i].getValue();
                        Log.d("http01", "Http request: Name--->" + name + ",Value--->" + value);
                    }
                    
                    try {
                        HttpResponse httpResponse = httpCient.execute(httpGet);
                        
                        //获取响应头，并用Header数组接收
                        Header [] responseHeaders = httpResponse.getAllHeaders();
                        //遍历Header数组，并打印出来
                        for (int i = 0; i < responseHeaders.length; i++) {
                            String name = responseHeaders[i].getName();
                            String value = responseHeaders[i].getValue();
                            Log.d("http01", "Http response: Name--->" + name + ",Value--->" + value);
                        }                    
                        
                        if (httpResponse.getStatusLine().getStatusCode() == 200) {
                            HttpEntity entity = httpResponse.getEntity();
                            String response = EntityUtils.toString(entity,"utf-8");//将entity当中的数据转换为字符串
                            
                            //在子线程中将Message对象发出去
                            Message message = new Message();
                            message.what = SHOW_RESPONSE;
                            message.obj = response.toString();
                            handler.sendMessage(message);
                        }
                        
                    } catch (Exception e) {
                        // TODO Auto-generated catch block
                        e.printStackTrace();
                    }
                    
                }
            }).start();//这个start()方法不要忘记了        
        }
    }

