---
layout: post
title: "Linux学习基础篇02 ubuntu的配置(一) 启用root登录界面"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-02 09:01
---

> 前面一章，介绍了如何安装ubuntu系统。这一章开始，我们将陆续介绍一些ubuntu的配置。

> 默认情况下，在ubuntu的GUI登录界面，你是无法直接通过root登录的。如果你想用root登录到ubuntu的GUI系统中，通过以下操作即可实现。

## 1. 设置root账户和密码

**第一步：用普通账户登录。**

**第二步：登录之后，通过以下命令设置root账户密码：**

    $ sudo passwd root

**第三步：设置root密码成功后，在终端可以通过以下命令切换到root：**

    $ su
或

    $ su -

 
## 2. 开启root登录界面

**第一步：编辑配置文件lightdm.conf，开启登录界面输入选项。**

编辑文件/etc/lightdm/lightdm.conf，可以采用以下命令：

    $ sudo gedit /etc/lightdm/lightdm.conf

在文件末尾，添加以下内容：

    greeter-show-manual-login=true

**第二步：通过reboot重启电脑**

    $ sudo reboot

**第三步：在登录界面就输入用户名(root)和密码进行登录了。**

