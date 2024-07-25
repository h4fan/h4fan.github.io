---
layout: post
title:  美国安全意识培训公司招聘到了朝鲜黑客？
tags: [社工,hack]
---

在看toot时，发现Krebs转发了一篇文章，KnowBe4公司招人招到了一名朝鲜黑客，并且声称朝鲜黑客要渗透他们，不过被他们发现了，并且KB公司通知了FBI。
原文链接在此：https://blog.knowbe4.com/how-a-north-korean-fake-it-worker-tried-to-infiltrate-us

文中指出，由于目前FBI还在调查中，因此披露的细节有限。因此本文是基于文章做的猜测。

## 美国安全公司
KnowBe4是美国一家安全意识培训公司，主要提供钓鱼模拟演练和安全意识培训。Mitnick曾经是这家公司的Chief Hacking Officer.

## 朝鲜黑客
网络上的新闻包括，朝鲜黑客攻击虚拟货币、攻击银行、社工安全研究员等。  
朝鲜黑客攻击一家美国安全意识培训公司的目的是什么？看看哪家公司的员工安全意识比较弱，然后发动定向钓鱼攻击？

## 细节猜测
故事是这样的，KnowBe4（下文称KB公司）想要为内部的人工智能团队招一名软件工程师，招聘岗位为首席软件工程师（Principal Software Engineer），职级还是很高的。  
KB公司发了招聘需求之后，收到了一些简历。  
接下来是面试，KB公司的HR团队在不同场景下对朝鲜黑客（化名：小金）进行了4次视频面试，同时HR确定了面试对象的照片和简历上是一致的。
```
小金通过了Principal Software Engineer的面试，看来小金的水平很强。

```
面试通过之后，进行背调、推荐验证等其他入职前的检查。  
由于小金使用的是偷来的身份，这个身份是一个真实的美国身份，因此他通过了检查。这里的照片展示的是小金偷来的身份（化名：山姆）和小金简历上的照片，照片通过deepfake进行了加强，看起来确实很像。
![stolen identity](/static/img/sam-kim.png)
有意思的是，网络上搜索山姆的照片，出现在多个网站上，并且名字还是不一样的。使用小金的照片进行搜索，相似照片里面还能搜到山姆的照片。
![sam search](/static/img/sam-search.png)
![sam search 2](/static/img/sam-search2.png)
![sam 2](/static/img/sam2.png)
![sam 3](/static/img/sam3.png)
![kim search](/static/img/kim-search.png)

之后KB公司雇佣了小金。

雇佣之后，KB公司将公司的Mac workstation寄给了小金（国外公司不少公司可以远程办公）。  小金的收货地址写的是一个“电脑农场”，美剧里面在东南亚就有很多这种农场，里面有很多当地的便宜工人来操作。  

IT农场的工人在电脑上安装软件，估计是远程控制之类的软件，小金通过远控接入进来。并且小金在夜晚工作，这样就是美国时间。真正的编码工作，估计是外包给其他人来做了。小金估计就是以此为跳板，获取权限，进一步的渗透。

在小金收到Mac后不久，KB公司的EDR软件就检测到恶意软件，告警通知到KB公司的SOC团队。

SOC团队检测到风险之后，联系小金进行调查，看看是什么问题。小金说他在根据路由器指南调试一个网速问题，可能导致被黑了。
SOC团队分析发现小金电脑上的一系列可疑行为，例如小金修改了一些session文件，传输了一些有害文件，运行一些未授权的软件。并且小金使用树莓派下载恶意软件。（树莓派在这里有什么作用？将恶意软件放在树莓派上，然后在树莓派上对Mac进行访问？或者在Mac上从树莓派上下载恶意软件？）。
SOC团队电话联系小金想了解更多细节，小金说不方便接电话，后面小金就不回消息了。  
SOC团队发现小金电脑上的一系列可疑行为之后，经过分析判断，认为这些行为可能是有意的，因此怀疑是Insider Threat或Nation State Actor。SOC团队的这个安全意识很强。  
之后SOC团队就把小金的设备进行了隔离。
SOC团队将发现的风险发给在Mandiant（安全公司）的朋友以及FBI。最终经过调查，发现小金是一个朝鲜黑客，伪装为美国程序员。

## 推断
* `References potentially not properly vetted. Do not rely on email references only.`
邮件推荐不够完善。一种可能是伪造邮件，一种可能是伪造发件人，还有一种可能是发件人邮件被黑了。

* `Sophisticated use of VPNs or VMs for accessing company systems`
使用了是KB公司的VPN？或者安装虚拟机运行VPN软件？

* 视频面试怎么通过？
Deepfake里面可以实时的做face swap，所以是可以做到的。还有一种可能，找一个比较像的人，然后开一点美颜之类的，估计也会比较像。说到小金偷了个美国身份，这个不就是美剧里面，Special Agency为Agents安排的Cover身份嘛。

这一下，案例库里面又多了一个社工的案例。
