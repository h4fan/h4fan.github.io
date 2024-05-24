---
layout: post
title:  AIChatProxy - 一个页面同时和3个AI通信
tags: [ai,chatbot]
---
 

# 问题
之前使用ChatGPT时，网页版是可以免费使用，API是收费的，之后出现了很多拿网页token去调用后端的Proxy。之前的想法是，能否直接在浏览器里面调用？  
最近在用国内多个AI时，发现在一个ai里面总是搜不到满意的答案。思考能否同时调用多个AI，这样可以进行答案对比，也能看到多个不同的结果。我不想分别到各个AI的网页中复制粘贴问题，并且我也没有买API，有没有自动化的办法呢？
答案是有的。
本文呈现一个方法，AIChatProxy，可以在一个网页中调用3个不同的AI来获取答案，分别是混元、千问、ChatGLM。
# demo
![demo](/static/img/aichatproxy1.png)

# online demo
https://aichatproxy.pages.dev/  
需要下载online demo的插件。

# 原理
通过在AI webpage中增加javascript，这里使用的是浏览器插件的方式插入，也可以通过手工控制台粘贴或者油猴脚本插入。  
通过javascript监听message，接收来自网页端的prompt语句。使用window.opener.postMessage将结果传回给网页端进行展示。即可实现2个页面的双向通信。

# 下载地址
https://github.com/h4fan/AIChatProxy
