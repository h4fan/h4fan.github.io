---
layout: post
title:  airtag神奇的find my network
tags: [tech,bluetooth]
---

最近看的一个新闻，荷兰记者利用蓝牙追踪器定位到了一艘防空护卫舰的位置，成功掌握了24小时的记录。

## Bluetooth tracker
[Bluetooth tracker hidden in a postcard](https://www.tomshardware.com/tech-industry/cyber-security/bluetooth-tracker-hidden-in-a-postcard-and-mailed-to-a-warship-exposed-its-location-a-eur5-gadget-put-a-eur500-million-dutch-ship-at-risk-for-24-hours)文章里面提到，像airtag这样的蓝牙追踪器就可以做到类似功能。

## 疑问
比较令我好奇的是，这个追踪器是如何传回位置数据的？这么远距离，wifi也连不上。难道是内置手机卡，利用手机流量？之前倒是有看到新闻说利用树莓派来追踪货物的位置的。

## airtag
新闻里面提到了airtag，就去deepseek里面搜索了一下工作原理，发现，还是很神奇的。相当于利用了庞大的apple设备网络。

苹果官网的一段介绍如下：
```
亿万无名英雄，一起帮你找。
如果你的东西落在咖啡馆、海滩或机场等地方，“查找”网络可以利用超过 10 亿个iPhone、iPad 和 Mac 设备来帮你找，同时每一步都力保你的隐私安全。

它怎样发挥作用？
你的 AirTag 会发出安全的蓝牙信号，让“查找”网络中在它附近的设备可以侦测到。这些设备会将你 AirTag 的位置发送至 iCloud，然后你就可以在查找 app 的地图上看到它了。整个过程都是匿名的，并经过加密，以保护你的隐私安全。与此同时，查找也很高效，因此无需担心电池续航或数据流量的消耗。
```
原理也说出来了，就是airtag的蓝牙信息，apple的“查找”网络里面的设备，是可以接收到这个设备的信息，然后将位置报告给iCloud。原理非常简单，因为我们的设备，开了蓝牙之后，是可以不断扫描周围的蓝牙设备的，扫描之后，再加上位置信息，传给服务器。

基本上，现在很多设备的蓝牙是一直开着的，要连接手环、手表、蓝牙耳机等设备，所以蓝牙开启是很容易的。并且苹果设备似乎会自动开启蓝牙。

还有一个类似的现象，就是有些手机，会利用wifi来不断扫描周围的热点，比如某些wifi钥匙，会将wifi名字和位置信息上传。
## find my network
真正神奇的是find my network，因为相当于airtag周围的苹果设备，都会充当一个中继。考虑到现在苹果手机普及率这么高，周围有一台苹果设备的概率还是非常高的。

## evil ideas
疑犯追踪里面，有一个场景，就是Finch在一个摄像头看不到地方，和the machine对话，问machine是否知道它在哪，结果超出了Finch的预料。因为machine黑了一些手机和笔记本电脑设备，利用这些设备的摄像头，还有一些反光信息，定位到了Finch。

现在让我们来假设一下，如果某个神奇的AI黑了某个手机系统服务商或者某个大规模部署的应用，是不是就能定位到任意的蓝牙设备了？

当然，功能还得稍微修改一下，毕竟是神奇的AI，改一下代码还是没有问题的。
```
隐私保护全内置。你的 AirTag 在哪里，只有你自己能看到。你的位置数据和历史记录绝不会存储在 AirTag 中。传递你 AirTag 位置数据的设备也始终保持匿名，而且位置数据在查找的每一步都有加密保护。因此，就连 Apple 也不知道你 AirTag 的位置，以及帮忙找到它的设备是何身份。
```
