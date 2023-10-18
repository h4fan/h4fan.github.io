---
layout: post
title:  Kali Linux虚拟机硬盘空间不够了怎么办？
tags: [pve,qemu,kali,disk]
---

# Kali Linux虚拟机硬盘空间不够了怎么办？

Kali Linux一直跑着，没怎么管它，最近执行一个命令，提示没有空间了。

装了几个Java环境，估计是各种依赖包，删也不知道该从哪里删。本来想放着，今天发现pve的界面上有增加磁盘空间的功能，抱着试一试的心态，试试看。 

# 挤空间
看着n个虚拟机，删了几个虚拟机。最后一算，发现硬盘上本身还有不少空间的。闹了个乌龙！ 

# 点击增加磁盘空间
在pve界面上增加磁盘空间，之后启动kali，执行`df` 发现还是百分之90几，看来不能神奇的自动增加。

# fdisk -l
`fdisk -l`，发现增加的空间已经是有了，接着就是怎么把它显示出来。

# 分区
```bash
fdisk /dev/sda
m 
l 
n 
回车
回车
w
```
# 格式化
```
fdisk -l
mkfs -t ext4 /dev/sda3(替换为你刚刚增加的分区)
```

# mount
```
mkdir /media/sda3
mount /dev/sda3 /media/sda3
```

# 永久挂载
编辑 `/etc/fstab`，最后增加一行
```
/dev/sda3	/media/sda3		ext4	defaults	0 	0

```
重启
# 修改权限
由于/media/sda3是owner是root，还需要改成自己的用户，方便读写。
```
sudo chown kali /media/sda3
```

# 校验
执行`df`，发现已经成功挂载，并且当前用户可以读写。ok. 成功给Kali Linux虚拟机增加了磁盘空间。


### 参考
https://blog.csdn.net/m0_49864110/article/details/129097305
https://blog.csdn.net/JangBingYang/article/details/127781868
