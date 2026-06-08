---
layout: post
title:  学习Device Bound Session Credentials
tags: [security,cookie]
---

Device Bound Session Credentials，通过使用设备端的Private Key来验证当前设备与Session Cookie的关系，削弱Cookie窃取的危害。

# Device Bound Session Credentials
需要改造登陆过程，可查看[Chrome DBSC](https://developer.google.com/docs/web-platform/device-bound-session-credentials)相关页面。

登陆成功之后，服务端发送Secure-Session-Registration头，Chrome浏览器在本地进行密钥注册，之后将公钥发送给服务端，服务端需保存。

当客户端Cookie过期之后，Chrome会发起Cookie refresh，服务端返回Challenge，客户端Chrome会利用机器中的Private Key对Challenge进行签名等操作，服务端再进行验证，验证通过之后返回新的Cookie。

Chrome DBSC也有提示，如果malware在Session注册阶段就存在了，那么malware是可以获取到Private key的，这样后面的签名也是可以自己完成的。

# 缩短攻击时间窗口
DBSC可以显著缩小Cookie窃取之后的攻击时间窗口。

因为认证的cookie是一个短期Cookie，那么在这个短期Cookie过期之后，是需要新的Cookie的，而攻击者没有Private key的情况下，无法完成Cookie Refresh，所有没法长时间的利用这个Cookie。

# 原理
通过增加了本地电脑中的Private key，来增加了一个新的认证因素。

传统情况下，拿到cookie之后，换台电脑，仍然可以登陆，使用DBSC之后，在cookie有效期内，也依然可以登陆，所有DBSC不解决cookie有效期之内的攻击。

但是如果cookie过期了，如果是在本机上，那么Chrome就可以自动执行cookie刷新的操作，其中关键的一点，是服务端会发来一个challenge，客户端Chrome需要能够证明自己，而证明的关键点就是拥有private key，可以对challenge进行签名，由于服务端有该session对应的public key，因为可以对客户端发来的请求进行验证。而攻击者，通常情况下，只窃取了cookie，没有private key，所有攻击者没法刷新cookie。这样cookie过期之后，之内重新登陆了，无法进一步利用。

而private key由于Chrome生成之后保存到了本机安全硬件模块中，攻击者通常更难获取到private key。不过正如前面的提示所说，如果注册阶段就存在malware，那么malware是可能获取到private key的。


