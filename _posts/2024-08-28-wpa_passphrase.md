---
layout: post
title:  the 'bad' psk
tags: [tech,pve]
---

最近路由器的wifi名字改了，但是密码没有改。
看过我博客的朋友可能知道，我之前的[pve](https://h4fan.github.io/2021/06/09/%E5%A6%82%E4%BD%95%E7%BB%99%E6%B2%A1%E7%BD%91%E7%9A%84%E5%8F%B0%E6%9C%BA(pve)%E8%A3%85rtl8821cu%E6%97%A0%E7%BA%BF%E7%BD%91%E5%8D%A1%E9%A9%B1%E5%8A%A8.html)是用的wifi无线连接的。
自然我就上不了网了。

## 寻找原因
遵循能够运行就不要改动的原则，我已经不记得我是如何配置的了。  
赶紧查看我的pve的博客，找到命令。找到配置文件，修改ssid，运行
`wpa_supplicant -B -i <wificard> -c /etc/xxx.conf`的命令，成功，连上了wifi，原来修改如此简单，重启一下。

咋又没有网了？  
赶紧插上显示器，连上键盘去看一看。  
iwconfig，发现wifi没有连成功。再次手动运行命令，又成功了。

再重启，发现还是不行。  
难道是路由器里面改了配置，把我的mac屏蔽了？

换了一个新的ssid，依然是自动连接不成功，手动连接成功。  
中间问了chatglm和doubao，坑d啊，给的还是错误的命令，执行报错。

以为是没有开机自动启动，但是这个说法不对，因为之前一直是好的，能够正常运行。  
`systemctl status wpa_supplicant`，显示loaded(inactive)。

再次去ai里面搜索，怎么添加开机启动。发现不止wpa_supplicant这个服务，还有一个wpa_supplicant@\< wificard>名称的服务。
`systemctl status wpa_supplicant@\< wificard>`，一看，说是密码不对。

我就奇怪了，密码没有改，怎么就不对了？

wpa_supplicant-<wificard>.conf里面的密码是这样的
```
network={
    ssid="MYSSID"
    #psk="passphrase"
    psk=59e0d07fa4c7741797a4e394f38a5c321e3bed51d54ad5fcbd3f84bc7415d73d
}
```
而/etc/xxx.conf里面是这样的
```
network={
    ssid="MYSSID"
    psk="passphrase"
   
}
```
密码怎么就不对？  
拿着wpa_passphrase生成密码，发现是一个新的psk，问题找到了。确实是密码不对。  
重新生成密码，更新psk的配置，重启，连接成功。  
测试的时候，使用的是明文的密码，但是配置文件里面使用的是生成的密码，所以测试能成功，重启之后连不上。

## wpa_passphrase
```
$ wpa_passphrase MYSSID passphrase
network={
    ssid="MYSSID"
    #psk="passphrase"
    psk=59e0d07fa4c7741797a4e394f38a5c321e3bed51d54ad5fcbd3f84bc7415d73d
}
```
为什么密码不对呢？因为wpa_passphrase生成的psk是和ssid有关系的，所以ssid改了，但是密码没有改，psk也是不一样的。

wpa_passphrase pre-computes PSK entries for network configuration blocks of a wpa_supplicant.conf file. An ASCII passphrase and SSID are used to generate a 256-bit PSK.

```
00061         pbkdf2_sha1(passphrase, ssid, os_strlen(ssid), 4096, psk, 32);
00062 
00063         printf("network={\n");
00064         printf("\tssid=\"%s\"\n", ssid);
00065         printf("\t#psk=\"%s\"\n", passphrase);
00066         printf("\tpsk=");
00067         for (i = 0; i < 32; i++)
00068                 printf("%02x", psk[i]);
00069         printf("\n");
00070         printf("}\n");
```
查看源码，我们 发现psk是pbkdf2_sha1生成的，而ssid是作为一个参数传入进去了。
```
pbkdf2_sha1 - SHA1-based key derivation function (PBKDF2) for IEEE 802.11i : ASCII passphrase : SSID : SSID length in bytes : Number of iterations to run : Buffer for the generated key : Length of the buffer in bytes Returns: 0 on success, -1 of failure

This function is used to derive PSK for WPA-PSK. For this protocol, iterations is set to 4096 and buflen to 32. This function is described in IEEE Std 802.11-2004, Clause H.4. The main construction is from PKCS#5 v2.0.
```

https://docs.ros.org/en/diamondback/api/wpa_supplicant/html/wpa__passphrase_8c_source.html
https://linux.die.net/man/8/wpa_passphrase
https://man.freebsd.org/cgi/man.cgi?wpa_passphrase

