---
layout: post
title:  unicode2utf8 - burp插件将unicode显示为中文
tags: [burpsuite,unicode]
---

在使用burp进行测试的时候，经常会碰到一些接口返回unicode字符串，直接看也不知道是什么，如下图：  

![image](/static/img/unicode2utf8_1.webp)

这个时候，就需要把unicode字符串复制出来，拿到浏览器console里面去显示。  
复制来复制去，有时候还是比较麻烦的。  
那么有没有插件能够直接在bp里面将unicode转换为utf8呢？  
有的。效果如下：
![image](/static/img/unicode2utf8_2.webp)

可以看到，\u unicode已经转为utf8了，显示的是中文。  
![image](/static/img/unicode2utf8_3.webp)

访问获取：
[https://github.com/h4fan/unicode2utf8](https://github.com/h4fan/unicode2utf8)
