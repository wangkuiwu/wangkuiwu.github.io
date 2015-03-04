---
layout: post
title: "Linux学习基础篇02 ubuntu的配置(四) 终端快捷键和常用配置"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-02 09:04
---

> 本文将讲解linux终端的常用快捷键和常用配置。

# 1. 终端常用快捷键

## 1.1 启动终端

快捷键是`Ctrl+Alt+T`。当然，你也可以修改启动终端的快捷键，该快捷键的定义位置如下：  
"System Settings"  -->  "Keyboard"  -->  "Shortcuts"  -->  "Launchers"  -->  "Launcher terminal"

## 1.2 终端内置快捷键

    Ctrl+C  取消当前任务
    Ctrl+D  终止
    Ctrl+E  将光标移到末尾
    Ctrl+A  将光标移到开头
    Ctrl+B  光标后退一个字符
    Ctrl+F  光标前进一个字符
    Ctrl+U  删除光标到开头的内容
    Ctrl+K  删除光标到开头的内容
    Ctrl+W  删除光标的前一个单词
    Ctrl+R  在终端的历史指令中查找
    Ctrl+P  上一个指令
    Ctrl+N  下一个指令


# 2. 终端常用配置

下面讲解~/.bashrc中的常用配置。

**2.1 修改终端标题**

当我们在终端进入的路径很深的时候，对应的在终端显示的路径也越长。这样很不利于观察！可以修改~/.bashrc中的PS1。

找到~/.bashrc中的PS1定义，将其修改为以下内容：

    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '

说明：  

    \u 表示当前用户
    \h 表示主机名称
    \w 表示绝对路径
    \W 表示路径的最后一个目录的名称
    


**2.2 添加自定义别名**

~/.bashrc中的别名是一个非常简单且实用的技巧。例如，在~/.bashrc中添加以下内容

    alias cdskywang='cd /home/skywang/mnt/partion3_ext4/skywang'

有了上面的别名之后，执行`cdskywang`就相当于执行了`cd /home/skywang/mnt/partion3_ext4/skywang`指令！



**2.3 添加自定义函数**

如果你熟悉shell脚本的话，你可以在~/.bashrc中定义一些常用的方法。然后，随时都可以调用这些方法。例如，我在自己的~/.bashrc中添加了编译&运行java文件的函数，内容如下。

    function jjava()
    {
        local file_name=$1
        local file=${file_name%\.*}
        shift 
        local args=$*
        case $file_name in
            "") 
                echo "Error : please input in a validate parameter."
                ;;  
            *)  
                javac $file_name && java $file $args
                ;;  
        esac
    }

有了上面的定义之后，就可以很方便的编译&运行java文件。假设存在一个java文件Hello.java，正常情况下，编译&运行Hello.java需要两条指令。

    $ javac Hello.java
    $ java Hello

有了jjava函数之后，可以直接通过以下指令来编译&运行Hello.java。

    $ jjava Hello.java

