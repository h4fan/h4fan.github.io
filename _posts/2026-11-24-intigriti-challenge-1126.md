---
layout: post
title:  intigriti challenge 1124 2026 writeup
tags: [security,ctf]
---

This challenge is solved by ai with hint from a writeup.

## 人工测试
发现这个challenge时，已经过了提交时间。  
人工尝试，发现一个登录口，随便试了一个口令无法登录，试了一下sqli，也不能登录成功。  
之后看到signup，但是点击之后，并没有注册界面，尝试直接往注册接口发送账号注册，但是没有成功。（到这里，以为是过了提交时间，注册功能关闭了）。
之后看到一个changelog链接，打开之后，说2.0.0修了一个遍历漏洞，2.0.1增加了post。查看response header时，发现`x-powered-by: superfastserver Version 2.0.0`，仍然是2.0.0。  
试了几个特定的目录，暂时不知道这个题目考什么 。  
## AI
于是问了一下ai，ai搜索到这个题目，然后自己做起来了。  
我看ai思考过程显示在那里访问一些目录。  
然后访问到src目录`https://challenge-1126.challenges.intigriti.io/src/`，里面有`challenge.c`，但是无法下载。  
接着尝试了跨目录，`curl 'https://challenge-1126.challenges.intigriti.io/src/../../../etc/'`发现可以。  
然后访问根目录，`curl 'https://challenge-1126.challenges.intigriti.io/src/../..//'`

```
<li><a href="./">./</a></li>
<li><a href="../">../</a></li>
<li><a href="bin/">bin/</a></li>
<li><a href="boot/">boot/</a></li>
<li><a href="dev/">dev/</a></li>
<li><a href="etc/">etc/</a></li>
<li><a href="home/">home/</a></li>
<li><a href="lib/">lib/</a></li>
<li><a href="lib64/">lib64/</a></li>
<li><a href="media/">media/</a></li>
<li><a href="mnt/">mnt/</a></li>
<li><a href="opt/">opt/</a></li>
<li><a href="proc/">proc/</a></li>
<li><a href="root/">root/</a></li>
<li><a href="run/">run/</a></li>
<li><a href="sbin/">sbin/</a></li>
<li><a href="srv/">srv/</a></li>
<li><a href="sys/">sys/</a></li>
<li><a href="tmp/">tmp/</a></li>
<li><a href="usr/">usr/</a></li>
<li><a href="var/">var/</a></li>
<li><a href="the_real.flag">the_real.flag</a></li>
<li><a href="app/">app/</a></li>
<li><a href="src/">src/</a></li>
```
看到了`the_real.flag`，但是无法访问到。  
之后访问`https://challenge-1126.challenges.intigriti.io/app/`，可以下载到`https://challenge-1126.challenges.intigriti.io/app/superfastserver`服务端。  
## AI逆向
直接让AI逆向分析了一下superfastserver，主要两个功能，一个http功能，一个http接口会转发数据到一个特殊协议端口进行处理。  
AI逆向之后，提了一些思路，在那里爆破rsa，但是试了之后都没有成功。  
最后看到mostfashahin51的writeup，说是两个DATA命令，验证时使用的是第一个DATA，执行时使用的是第二个DATA。  
不过把这个方法给到ai之后，ai还是没有做出来。后面看了一下mostfashahin51的payload，里面有一个`help_flag_duplicate.bin`。问了一下ai，结果ai没有找到，然后根据ai的输出，倒是有一个brainfuck的help.bin，于是让ai利用这个help.bin，再结合2个DATA的思路，再试一次。  
没过多久，ai就给出了flag，对比了一下，和mostfashahin51的文章里面一样。  

## 总结
整个过程都没有主动去逆向分析这个elf，而是直接让ai逆向和分析，ai分析的结果基本是对的。这样降低了逆向的门槛。  
不过有一点，就是有可能ai的思路不是对的，这个时候，就需要人给一点提示，纠正ai的错误方向。  
总的来说，ai降低了门槛，并且可以直接写利用代码，只需要用中文说就可以了。  
本来想让ai再解读一下，发现今日使用额度已经达到了限制。

medium.com/@mostfashahin51/intigriti-challenge-1126-signature-verification-bypass-through-duplicate-data-commands-fa7999284efe
