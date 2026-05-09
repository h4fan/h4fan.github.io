---
layout: post
title:  为什么cloudflareinsights beacon js出现在网站上了
tags: [tech,security]
---

为什么cloudflare 的Web Analytics自动添加在了网页中？

## problem
最近开发了一个html应用，部署在cloudflare pages上进行测试，查看网页源代码时，出现了一段Pages Analytics代码。

```
<!-- Cloudflare Pages Analytics -->
script defer="" src="https://static.cloudflareinsights.com/beacon.min.js/..."
```
为什么这段代码引起了我的好奇？

因为，这个网站，添加了自定义域名。使用pages默认域名访问时，没有这段Analytics代码，但是使用自定义域名的时候，就出现了这段Analytics代码。

并且，我的另外一个站点，添加的是clarity的统计代码，发现页面除了有clarity统计代码，也出现了这段Analytics代码。

## Web Analytics
我们知道cloudflare pages里面有Web Analytics功能，刚开始以为是自己打开了开关。结果发现两个站点都没有开启Analytics。

难道是cloudflare 自己默认添加的？

## 搜索
带着疑问，我们去进行网络搜索，找到一个reddit（/r/CloudFlare/comments/1rr9mzs/cant_stop_cloudflare_from_injecting_beaconminjs/），这个用户遇到了同样的问题，并且提供了解决办法。不过由于cloudflare的坑人UI，我并没有找到这个功能在哪里关闭。

接着又找到一个网站用户（https://burgeonlab.com/blog/cloudflare-web-analytics-rum-injected-tracking-beacon-script-into-my-sites/）,也遇到了类似问题，不过这个用户给出了截图。

抱着试一试的心态，再去cloudflare里面找了一下，点击域名下面的`analytics&logs` 中的`web analytics` ，在页面中部的左下方找到`quick actions`，里面有个 `manage rum settings`设置，点击之后修改设置即可。为什么说UI坑人呢？因为这个UI和下面的文档链接一样，给我的感觉就是一个配置说明或者文档链接，根本不是可以点击的按钮。不知道这个UI是有意还是无意的，如果是有意的，那有点坏。

## 总结
所以原因是，默认在域名级别开启了统计，所以使用自定义域名访问时，都会被插入一段beacon代码，无论你的网站是否开启了Analytics。

问了DeepSeek，给出了类似的结论，` Cloudflare 账户全局启用了 真实用户测量 (RUM, Real User Measurements) 功能，它可能会覆盖站点的单独设置`。

另外一点，Cloudflare的Web Analytics真的很鸡肋，根本提供不了什么有用的信息。
