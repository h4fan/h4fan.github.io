---
layout: post
title:  修复被更新坏了的系统
tags: [linux]
---

打开一台Linux虚拟机，更新完系统之后，无法进行系统了。

# update
打开Linux虚拟机，`apt update` `apt upgrade`，更新了一下系统。
于是一边做测试，一边让系统自己更新。临近吃饭的时候，还没有更新完，于是让系统自己更新。
结果吃完饭回来，系统卡在一个界面黑色界面上，只有一个线在闪动。于是强制关机，结果启动时报错了，无法进入系统。

接下来就开始问ai怎么处理。
# fix
## recovey
首先尝试进入recovery，结果在recovery的界面，也显示`rsyslog.service`启动失败，无法进入系统。

## single
在启动界面，按`e`,编辑参数，在linux开头的那一行，修改`ro quiet splash`为`ro single`,按ctrl+x，进入系统，系统提示root 账号被锁了。

## 修改root密码
继续在启动界面，修改为`ro single systemd.mask=rsyslog.service`，进行系统，得到`root@(none):/#`shell。
```
mount -o remount,rw /
passwd root
# 修改root密码
exec /sbin/init
#重启系统
```
## 查看错误

```
systemctl status rsyslog.service
journalctl -xeu rsyslog.service
rsyslogd -N1

```
查看到错误是`code=killed, signal=SEGV`
发现错误是`/usr/sbin/rsyslogd: Relink /usr/lib/x86_64-linux-gnu/libfastjson.so.4' with/usr/lib/x86_64-linux-gnu/libm.so.6`

提示说是要降级libfastjson，`sudo apt-get install libfastjson4=0.99.9-1`
但是找不到`0.99.9-1`版本的libfastjson4

## apt
apt install 一直提示umet dependency, `apt --fix-broken install`以及`apt clean`和`apt autoclean`之后，仍然无法成功。

## 网络
突然发现无法ping通网络，没有网了，`ifconfig` `iwconfig`查看，运行`systemctl start NetworkManager`，有ip了。
域名还不能解析，于是`echo 'nameserver 1.1.1.1' > /etc/resolv.conf`，添加dns。

终于有网了，但是apt install 时仍然`unmet dependency`，于是按照ai说的，删除本地的包列表索引，`sudo rm -rf /var/lib/apt/lists/*`，仍然不成功，一直抱python3-pyspnego这个包错误。

于是直接编辑status文件，`sudo vi /var/lib/dpkg/status`，使用/搜索pyspnego.
找到这种格式，删除掉。
```
Package: python3-pyspengo
Status: install ok half-configured  # 或者类似的状态
```

完了之后，继续update等操作，不知道是什么问题，可以update了，但是又出来一个新的错误。`sysusers.d/rtkit.conf unknown modifier u!`
又是一通clean update 等操作，不知道为啥，update成功了，最后终于进入了系统。

## mysql
进行系统之后，发现`python no module named dnslib`，pip安装提示包是被第三方管理的，但是apt安装又找不到这个包。
`pip install mysql-connector-python`无法成功，根据提示，使用命令绕过，`pip install  --break-system-packages -i https://mirrors.aliyun.com/pypi/simple/ mysql-connector-python`，最终成功安装，脚本启动。


最终，系统启动成功。


