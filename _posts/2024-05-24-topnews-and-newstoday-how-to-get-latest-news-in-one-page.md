---
layout: post
title:  TopNews和NewsToday-一个页面中查看你关注的所有站点
tags: [ai,chatbot]
---
 

# 问题
关注了好几个科技和安全的站点，每日查看。想着能不能做个页面，可以直接查看最新的新闻，这样不用依次打开各个网站。（这个需求rss可以满足，不过有的页面没有rss或者rss不是最新的）

# demo
[TopNews](/topnews)
![topnews](/static/img/topnews.png)
[NewsToday](/newstoday)
![newstoday](/static/img/newstoday.pnd)


# 方法
方法非常简单，就是做一个接口，分别去获取最新的页面信息，并提取关注内容，返回给前端进行展示。
## TopNews
首先做了[TopNews](/topnews)，就是直接拉起最新的新闻，进行展示。
## NewsToday
在每天查看TopNew的过程中，发现有几个源更新比较慢，每天在TopNews中看到的都是相同的内容，所以做了个[NewsToday](/newstoday)，将TopNews中的链接按日期保存下来，重复的就去掉，这样就可以获取到每天新增的新闻。  
使用CF的Cron定时触发一直不成功，采取了个临时的方法，如果NewsToday返回为空，则跳转到TopNews进行查看。

