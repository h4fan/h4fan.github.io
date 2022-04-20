---
layout: post
title:  使用python扩展burp extender
tags: [burp]
---

之前写了几个python插件，记录一下想法。

# burp
burp是java写的，用java来写burp插件肯定是更匹配的。但是我更喜欢用python来写插件。用python来写插件有一个问题，那就是必须使用Jython。然而现在只有Jython2.7，python都已经3.10了，而Jython还不知道什么时候能有3点几。这样就会在写插件时api会受限，比如requests库用不了，得使用原生库发http请求，总之，受限很多。后来发现可以下载一些库的jar包，可以一起使用，但是要使用sqlite时，发现高版本的有问题，就得不断往下降版本去找可用的库，也比较麻烦。
后面想到做rpc，直接在本地开一个rpc server，这样在burp里面调用即可，这样就可以在rpc server中任意使用python3了。在代码中使用的是jsonrpc的一个python库。
另一个点是bp太吃内存了，有多少吃多少，特别是开一个自动化扫描时，时常会因为内存问题最后崩了，这样就看不到结果。所以需要将结果保存出来。
还有一块是做了个被动的检测，比如插件里面做了一些扫描，将扫描结果记录到本地数据库，同时根据查询远端接口比如dnslog的结果，将匹配的扫描显示出来。这里通过继承burp extender，可以自己实现一点小功能。

# extender
有了前述基础，就可以做ssrf的扫描。比如前几天的扫描，中间关闭了bp，后面打开时，仍然可以在dnslog收到之前的请求时，在bp里面展示结果。
