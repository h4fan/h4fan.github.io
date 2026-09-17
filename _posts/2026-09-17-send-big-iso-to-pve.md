---
layout: post
title:  用U盘将大ISO拷贝到PVE
tags: [tech,pve]
---

记录一下如何将大ISO文件拷贝到PVE上面。

## 问题
需要在PVE上面安装一个新的OS，选择了Ubuntu。下载Ubuntu.ISO之后，发现文件有6G大小。  
使用PVE的web界面上传ISO，发现速度慢得可怜，需要十几个小时才能上传完成。  
使用wget从本机下载过去也非常慢，于是考虑用U盘传输。

## FAT32
插入U盘，复制ISO文件，结果提示文件过大。  
原来U盘是FAT32，最大文件不支持6G。  
于是压缩iso，发现压缩之后，还是6点几G。（deepseek提示iso已经是压缩的文件，再次压缩提升不了多少） 

## 分块
问了DeepSeek，给了几种方案，一是格式化U盘，修改U盘格式；二是分块。  
最终选择分块。
使用`split -b 3000m ubuntu.iso part_`分块得到part_aa、part_ab、part_ac 3个文件。  
将3个文件复制到U盘中。

## 挂载U盘
在pve上面使用`lsblk`，得到u盘的设备名`/dev/sdb1`。  
之后mount，`mount /dev/sdb1 /mnt/usb`  
之后查看`ls /mnt/usb`，发现part文件。

## 复制
`cat /mnt/usb/part_* > ubuntu.iso`将3个part合并到iso

## iso image
`mv ubuntu.iso /var/lib/vz/template/iso`，查看web界面，发现已经可以看到ubuntu.iso的镜像了。

## 弹出u盘
`umount /mnt/usb`
