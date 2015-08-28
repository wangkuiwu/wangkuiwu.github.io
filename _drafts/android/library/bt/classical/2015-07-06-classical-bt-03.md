---
layout: post
title: "Android中的经典蓝牙(三) 蓝牙服务器和客户端通信示例"
description: "android"
category: android
tags: [android]
date: 2015-07-06 09:03
---

> 本文介绍Android经典蓝牙实现的服务器和客户端通信的示例。

> **目录**  
[1. 示例概述](#anchor1)  
[2. 客户端详解](#anchor2)  
[3. 服务端详解](#anchor3)  

<a name="anchor1"></a>
# 1. 示例概述

## 1.1 示例说明
本示例包括两个蓝牙程序：客户端程序和服务端程序。需要准备两台手机，分别作为客户端和服务器；通过手动连接之后，即可以进行通信(相互发送消息)。

![img](/media/pic/android/library/bt/classical_bt/bt3_01.jpg)

说明：上面是客户端和服务端通信的大致流程。 


## 1.2 示例源码

点击查看：[客户端源代码](https://github.com/wangkuiwu/android_applets/tree/master/library/bt/classical_bt/02_client)

点击查看：[服务端源代码](https://github.com/wangkuiwu/android_applets/tree/master/library/bt/classical_bt/02_server)


## 1.3 效果图

![img](/media/pic/android/library/bt/classical_bt/bt3_02.jpg)


<a name="anchor2"></a>
# 2. 客户端详解

## 2.1 蓝牙连接

    private static final UUID S_UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB");

    // 根据mac地址获取蓝牙设备
    BluetoothDevice mmDevice = mBluetoothAdapter.getRemoteDevice(mac);

    ...

    @Override
    public void run() {
        BluetoothSocket mmSocket = null;
        // 获取客户端的Socket
        try {
            mmSocket = mmDevice.createRfcommSocketToServiceRecord(S_UUID);
        } catch (Exception e) {
            Log.e(TAG, "bound failed", e);
            return ;
        }

        // 蓝牙连接(连接到服务端)
        try {
            mmSocket.connect();
        } catch (IOException e) {
            try {
                mmSocket.close();
            } catch (IOException e2) {
                Log.e(TAG, "unable to close() socket during connection failure", e2);
            }
        }
    }

说明：  
(1) 首先是根据蓝牙的mac地址获取BluetoothDevice对象，即得到远程的蓝牙设备。  
(2) 然后调用createRfcommSocketToServiceRecord()创建客户端的socket，创建成功的话就会成功返回soeckt对象。  
(3) 接着通过socket调用connect()建立连接。若连接建立成功的话，则后面则可以收发数据。


## 2.2 蓝牙接收/发送数据的后台线程

    private class ConnectedThread extends Thread {
        private final BluetoothSocket mmSocket;
        private final InputStream mmInStream;
        private final OutputStream mmOutStream;

        public ConnectedThread(BluetoothSocket socket) {
            Log.d(TAG, "create ConnectedThread");
            mmSocket = socket;
            InputStream tmpIn = null;
            OutputStream tmpOut = null;

            // Get the BluetoothSocket input and output streams
            try {
                tmpIn = socket.getInputStream();
                tmpOut = socket.getOutputStream();
            } catch (IOException e) {
                Log.e(TAG, "temp sockets not created", e);
            }

            mmInStream = tmpIn;
            mmOutStream = tmpOut;
        }

        public void run() {
            byte[] buffer = new byte[1024];
            int bytes;

            // 不断监听输入(即服务器发过来的数据)
            while (true) {
                try {
                    // 从InputStream中读取数据
                    bytes = mmInStream.read(buffer);
                    Log.d(TAG, "RECEIVE: "+new String(buffer));
                } catch (IOException e) {
                    Log.e(TAG, "disconnected", e);
                    break;
                }
            }
        }

        /**
         * 发送数据给BT服务器
         */
        public void write(byte[] buffer) {
            try {
                // 将数据写入到OutputStream中
                mmOutStream.write(buffer);
                // 显示发送数据
                Log.d(TAG, "SEND: "+new String(buffer));
            } catch (IOException e) {
                Log.e(TAG, "Exception during write", e);
            }
        }

        public void cancel() {
            try {
                mmSocket.close();
            } catch (IOException e) {
                Log.e(TAG, "close() of connect socket failed", e);
            }
        }
    }

说明：ConnectedThread就是蓝牙后台收发数据的线程。  
(1) 在线程的run()方法中，会不断的监听服务端发送过来的数据。  
(2) 若需要向服务端发送数据，在调用write()方法即可。



<a name="anchor3"></a>
# 3. 服务端详解

## 3.1 蓝牙连接


    public class MainActivity extends Activity {
        private static final String TAG = "##skywang-Server";
        private static final int REQUEST_ENABLE_BT = 1;
        private static final UUID S_UUID = UUID.fromString("00001101-0000-1000-8000-00805F9B34FB");

        private String mMacAddress;
        private EditText mEtMsg;
        private TextView mTvMac;
        private TextView mTvInfo;
        private BluetoothAdapter mBluetoothAdapter = null;

        private RequestAcceptThread mServerThread = null;
        private ConnectedThread mConnectedThread = null;

        private Handler mHandler = new Handler() {
            @Override
            public void handleMessage(Message msg) {
                mTvInfo.append("\n"+(String)msg.obj);
            }
        };

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            setContentView(R.layout.main);

            // 初始化蓝牙
            initBt();

            // 编辑器
            mEtMsg = (EditText) findViewById(R.id.msg);
            // 发送消息
            findViewById(R.id.send).setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    String msg = mEtMsg.getText().toString();
                    if (mConnectedThread != null ) {
                        mConnectedThread.write(msg.getBytes());         
                    }
                    Log.d(TAG, "send message:"+msg);
                }
            });

            // 显示消息的TextView
            mTvInfo = (TextView) findViewById(R.id.tv_info);
            mTvInfo.setText("Not Connected!");

            ToggleButton tglButton = (ToggleButton) findViewById(R.id.tgl);
            tglButton.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
                @Override
                public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                    if (isChecked) {
                        Log.i(TAG, "start listen");
                        startListen();
                    } else {
                        Log.i(TAG, "stop listen");
                        stopListen();
                    }
                }
            }); 

        }

        /**
         * 初始化蓝牙
         */
        private void initBt() {
            // 获取默认的BluetoothAdapter
            mBluetoothAdapter = BluetoothAdapter.getDefaultAdapter();

            if (mBluetoothAdapter == null) {
                Toast.makeText(this, "Bluetooth is not available", Toast.LENGTH_LONG).show();
                finish();
                return ;
            }
        }

        /**
         * 检查BT是否打开，没有打开的话，则弹出打开BT确认对话框
         */
        private void checkBluetooth() {
            // 如果BT没有打开，则弹出"打开BT的提示窗口"
            if (!mBluetoothAdapter.isEnabled()) {
                Intent enableIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
                startActivityForResult(enableIntent, REQUEST_ENABLE_BT);
            }
        }

        private void startListen() {
            mServerThread = new RequestAcceptThread();
            mServerThread.start();
        }

        private void stopListen() {
            if (mServerThread != null) {
                Log.d(TAG, "interrupt server thread");
                mServerThread.interrupt();
                mServerThread = null;
            }

            if (mConnectedThread != null) {
                Log.d(TAG, "interrupt connect thread");
                mConnectedThread.interrupt();
                mConnectedThread = null;
            }
        }

        @Override
        public void onStart() {
            super.onStart();
            checkBluetooth();
        }

        @Override 
        protected void onDestroy() {
            super.onDestroy();
            stopListen();
        }

        private final class RequestAcceptThread extends Thread {
            private BluetoothServerSocket mBluetoothServerSocket = null;
            
            public RequestAcceptThread() {
                boolean bSuccess = false;
                BluetoothServerSocket tmp = null;

                try {
                    tmp = mBluetoothAdapter.listenUsingRfcommWithServiceRecord("bt_test_chat", S_UUID);
                    bSuccess = true;
                } catch (IOException e) {
                    Log.e(TAG, "listen failed", e);
                }
                mBluetoothServerSocket = tmp;
                mHandler.sendMessage(mHandler.obtainMessage(0, bSuccess ? "listen to socket" : "failed listen to socket!"));
            }

            @Override
            public void run() {
                super.run();

                BluetoothSocket socket = null;
                while(true) {
                    try{
                        mHandler.sendMessage(mHandler.obtainMessage(0, "waiting..."));
                        // 监听客户端的连接请求。这里是阻塞式监听！
                        socket = mBluetoothServerSocket.accept();
                        break ;
                    } catch (Exception e) {
                        mHandler.sendMessage(mHandler.obtainMessage(0, "accept exception"));
                        Log.e(TAG, "accept Error, socket:"+socket, e);
                    }
                }

                if(socket != null) {
                    mHandler.sendMessage(mHandler.obtainMessage(0, "connected success!"));
                    mConnectedThread = new ConnectedThread(socket);
                    mConnectedThread.start();
                }
            }

            public void cancel() {
                try {
                    mBluetoothServerSocket.close();
                } catch ( IOException e) {
                    Log.e(TAG, "cancel socket failed", e);
                }
            }
        }

        private final class ConnectedThread extends Thread {
            private OutputStream mOutputStream;
            private InputStream mInputStream;
            private final BluetoothSocket mmSocket;

            public ConnectedThread(BluetoothSocket socket) {
                mmSocket = socket;
                InputStream tmpIn = null;
                OutputStream tmpOut = null;

                try {
                    tmpIn = socket.getInputStream();
                    tmpOut = socket.getOutputStream();
                } catch (IOException e) {
                    Log.e(TAG, "connect thread error", e);
                }
                
                mInputStream = tmpIn;
                mOutputStream= tmpOut;
            }

            @Override
            public void run() {
                byte[] buffer = new byte[1024]; // buffer store for the stream
                int bytes; // bytes returned from read()

                while(true) {
                    try {
                        bytes = mInputStream.read(buffer);
                        mHandler.sendMessage(mHandler.obtainMessage(5, "RECEIVE: "+new String(buffer)));
                    } catch (IOException e) {
                        Log.e(TAG, "connect thread run error", e);
                        break;
                    }
                }
            }

            public void write(byte[] buffer) {
                try {
                    // 将数据写入到OutputStream中
                    mOutputStream.write(buffer);
                    // 显示发送数据
                    mHandler.sendMessage(mHandler.obtainMessage(0, "SEND: "+new String(buffer)));
                } catch (IOException e) {
                    Log.e(TAG, "connect thread write error", e);
                }
            }

            public void cancel() {
                try {
                    mmSocket.close();
                } catch (IOException e) {
                    Log.e(TAG, "connect thread cancel error", e);
                }
            }
        }
    }

说明：  
(1) MainActivity运行之后，会自动执行RequestAcceptThread线程。  
(2) 在RequestAcceptThread线程中，会先调用mBluetoothAdapter.listenUsingRfcommWithServiceRecord("bt_test_chat", S_UUID)来获取服务器的socket。注意**这里的S_UUID与客户端的S_UUID是一样的**。  
(3) 接着，它会在run()方法里面通过accept()等待客户端的连接请求。加入客户端请求成功的话，则跳转while(true)循环，然后新建并启动一个ConnectedThread线程。  
(4) ConnectedThread线程是服务端和客户端进行通信的后台线程。在ConnectedThread的run()方法中，会不断的读取客户端发给服务器的数据；同时，通过调用线程的write()方法可以向客户端发送数据。
