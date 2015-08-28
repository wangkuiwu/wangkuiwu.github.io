---
layout: post
title: "Android中的经典蓝牙(二) 基本示例"
description: "android"
category: android
tags: [android]
date: 2015-07-06 09:02
---

> 本文给出Android中经典蓝牙的基本示例

> **目录**  
[1. 示例概述](#anchor1)  
[2. 示例详解](#anchor2)  


<a name="anchor1"></a>
# 1. 示例概述

## 1.1 示例说明
该示例是演示Android经典蓝牙的基本内容，包括：  
(1) 开关检测  
(2) 打开蓝牙  
(3) 蓝牙扫描  
(4) 蓝牙配对   
(5) 蓝牙连接   
(6) 获取已经配对的蓝牙设备  

## 1.2 示例源码

点击查看：[源代码](https://github.com/wangkuiwu/android_applets/tree/master/library/bt/classical_bt/01_scan_and_bound)


<a name="anchor2"></a>
# 2. 示例详解

## 2.1 权限设置

需要在manifest中添加BT相关的权限。添加内容如下：

    <uses-permission android:name="android.permission.BLUETOOTH"/>
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>

## 2.2 获取蓝牙适配器

    private void initBt() {
        // 获取默认的BluetoothAdapter
        mBluetoothAdapter = BluetoothAdapter.getDefaultAdapter();

        if (mBluetoothAdapter == null) {
            Toast.makeText(this, "Bluetooth is not available", Toast.LENGTH_LONG).show();
            finish();
            return ;
        }
    }

说明：mBluetoothAdapter是BluetoothAdapter对象，它是蓝牙适配器，代表了本机蓝牙设备。通过该适配器可以进行开/关蓝牙、进行蓝牙扫描等动作。

## 2.3 打开蓝牙

    private void checkBluetooth() {
        // 如果BT没有打开，则弹出"打开BT的提示窗口"
        if (!mBluetoothAdapter.isEnabled()) {
            Intent enableIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
            startActivityForResult(enableIntent, REQUEST_ENABLE_BT);
        }
    }
说明：checkBluetooth()的作用是进行蓝牙检测。如果蓝牙没有打开，则弹出"打开BT的提示窗口"。

## 2.4 读取已绑定的蓝牙设备

    final Set<BluetoothDevice> pairedDevices = mBluetoothAdapter.getBondedDevices();
    Log.d(TAG, "name:"+device.getName() + ", mac:" + device.getAddress());

说明：只要获取了mBluetoothAdapter对象，任何时候都可以调用getBondedDevices()来获取跟本机蓝牙已配对的蓝牙设别列表。


## 2.5 蓝牙扫描

    // 监听蓝牙扫描广播
    IntentFilter filter = new IntentFilter(BluetoothDevice.ACTION_FOUND);
    registerReceiver(mReceiver, filter);

    ...

    // 开始扫描
    mBluetoothAdapter.startDiscovery();

    ...

    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();

            if (BluetoothDevice.ACTION_FOUND.equals(action)) {
                // 获取扫描到的BT设备
                BluetoothDevice device = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);
                Log.d(TAG, "discovery device: name:"+device.getName() + ", mac:" + device.getAddress());
            }
        }
    };

说明：  
(1) startDiscovery()是开始蓝牙扫描。  
(2) 在开始扫描之前需要监听BluetoothDevice.ACTION_FOUND广播。因为蓝牙扫描的设备都是通过该广播发出的。  

## 2.6 停止蓝牙扫描

    private void cancelDiscovery() {
        // 若已经在扫描中，则先停止扫描
        if (mBluetoothAdapter.isDiscovering()) {
            mBluetoothAdapter.cancelDiscovery();
        }
    }

说明：若需要停止蓝牙扫描，则需要调用cancelDiscovery()停止扫描。


## 2.7 蓝牙配对

        // 根据mac地址获取蓝牙设备
        mmDevice = mBluetoothAdapter.getRemoteDevice(mac);

        ...

        @Override
        public void run() {
            try {
                if (mmDevice.getBondState() == BluetoothDevice.BOND_NONE) {
                    // 通过反射，调用createBond进行配对
                    mmDevice.getClass().getMethod("createBond").invoke(mmDevice);
                }
            } catch (Exception e) {
                Log.e(TAG, "bound failed", e);
            }
        }

说明：蓝牙配对就是调用BluetoothDevice的createBond()方法。这里是通过反射来实现的。此外，蓝牙配对操作应该在子线程中处理，而不要放到主线程中。


## 2.8 蓝牙配对

        // 根据mac地址获取蓝牙设备
        mmDevice = mBluetoothAdapter.getRemoteDevice(mac);

        ...

        @Override
        public void run() {
            // 2. 建立连接
            boolean bSuccess = false;
            int retries = 0;
            // 取消扫描
            mBluetoothAdapter.cancelDiscovery();
            // 尝试连接3次
            while (retries++ < 3) {
                try {
                    // 根据反射获取蓝牙Socket
                    BluetoothSocket mmSocket = (BluetoothSocket) mmDevice.getClass().getMethod("createRfcommSocket", new Class[] {int.class}).invoke(mmDevice, 1);
                    // 进行蓝牙连接
                    mmSocket.connect();
                    bSuccess = true;
                    // 连接成功，则跳出while循环。
                    break ;
                } catch (IOException e) {
                    Log.e(TAG, "connect failed retries: "+retries, e);
                    try {
                        mmSocket.close();
                    } catch (IOException e2) {
                        Log.e(TAG, "unable to close() socket during connection failure", e2);
                    }
                }

                try {
                    // 休眠100ms，再重新尝试连接
                    sleep(100);
                } catch (InterruptedException e) {
                    Log.e(TAG, "connect interruption ", e);
                }
            }
        }

说明：蓝牙连接需要在子线程中进行。BluetoothSocket是通过反射调用createRfcommSocket()方法实现的。

