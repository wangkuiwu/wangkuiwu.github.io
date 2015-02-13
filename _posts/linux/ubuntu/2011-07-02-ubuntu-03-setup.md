---
layout: post
title: "Linux学习基础篇02 ubuntu的配置(三) 开机自动挂载磁盘分区"
description: "linux ubuntu"
category: linux
tags: [linux]
date: 2011-07-02 09:03
---

> 如果你想ubuntu开机后自动挂载其他分区(如windows系统的相关分区)，可以在本文找到答案！


添加开机自动挂载分区需要修改/etc/fstab。那么，我先了解一下/etc/fstab，然后再说明如何去修改它。

## 1. /etc/fstab说明

linux在启动的时候，会逐行去检测/etc/fstab中的内容。如果/etc/fstab中的某一行是有效的挂载语句，则挂载该行的分区。/etc/fstab中标准的挂载语句如下：

    file_system  mount_point  type  options  dump  pass

**说明**：  
file_system: 设备名称。可以是磁盘号/UUID/Label。  
mount_point: 挂载点。  
type: 分区类型。如，linux分区一般为ext4，windows分区一般为ntfs或fat32。  
options: 挂载参数。一般为defaults。常用参数如下：  
> auto 开机自动挂载  
> default 按照大多数永久文件系统的缺省值设置挂载定义  
> noauto 开机不自动挂载  
> nouser 只有超级用户可以挂载  
> ro 按只读权限挂载  
> rw 按可读可写权限挂载  
> user 任何用户都可以挂载

dump: 磁盘备份。默认为0，表示不备份。  
pass: 磁盘检查。默认为0，表示不检查。

**示例**：

    UUID=c19b6c68-de72-41dc-9261-a2b5ec432555   /   ext4   errors=remount-ro  0  1

第一项是UUID，UUID是磁盘分区的一个id号。通过`sudo blkid`可以查看所有磁盘的uuid。  
第二项是/，表示将该分区挂载都系统的根目录。  
第三项是ext4，表示该分区是ext4格式。  
第四项是挂载选项，表示挂载为只读。  
第五项是0，表示不对分区进行备份。  
第六项是1，表示挂载分区时会分区进行坏块检测。



## 2. 添加开机自动挂载项

例如，将系统ntfs格式的/dev/sda3自动挂载到/home目录。编辑/etc/fstab，并添加如下内容即可：

    /dev/sda3  /home  ntfs  defaults  0  0

可以通过`sudo blkid`或者`sudo fdisk -l /dev/sda`等方式查看分区的格式。



