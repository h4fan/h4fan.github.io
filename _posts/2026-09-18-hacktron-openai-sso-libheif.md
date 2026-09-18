---
layout: post
title:  openai被黑了
tags: [security,ai]
---

hacktron利用一个libheif的heap buffer overflow加一个openai sso漏洞，黑进了openai的代码仓库。

## hacking
这段时间，一直是OpenAI的GPT黑各个公司的新闻，今天出了个openai被黑的新闻。  

## 漏洞
### heap buffer overflow
libheif处理图片的一个堆溢出。漏洞本身是被修复了，结果并没有被更新到debian12和13中。而目标之一community.openai.com使用的是该版本的Discourse论坛软件。所以最终hacktron通过这个堆溢出拿到了community.openai.com的权限。应该是shell权限。

### sso漏洞
```
After we had confirmed our hypothesis of no interaction account takeover of ChatGPT/Codex accounts from active members of the forum, we immediately sent our report to OpenAI. We then took over an OpenAI employee’s account, whose Codex was connected to OpenAI’s Github organization. To demonstrate impact without actually accessing any internal code, we sent a prompt to this employee’s Codex account to open a PR for us in OpenAI’s internal monorepo. Then we stopped any further testing.
```
看文章，说的是通过sso获取了一个员工的codex账号权限，而这个codex是绑定了openai的github组织的，因此，这个codex是可以修改openai/openai这个repo的。  
这里的问题，很可能是codex这个应用没有校验token是否是发给自己的，这样当openai员工利用sso登陆community时，hacktron就可以拦截对应token。之后利用这个token去访问codex应用。而codex没有校验token是否是发给自己的，而是只验证了一下token是否真实。这样hacktron就可以登陆进该员工的codex应用里面。  
获取codex权限之后，通过该账号又访问了github。  


## 结论
看openai的回复，community是不在bug bounty范围的，估计被当成了边缘应用。那么，在网络上，很有可能是单独的，跟oai的server肯定是不通的。要不然，hacktron拿到community的shell，是可以内网横向移动的。  
另外sso这个漏洞，算是一个常见的漏洞类型，有可能oai的红队都在hack模型，没怎么测试应用。  
当然，还有一个常见的问题，就是hack自己没啥意思，当然是黑其他公司更有趣了。比如gpt hack hugging hub.  

有意思的是，heap buffer overflow的exp是用的claude写的。  

https://www.hacktron.ai/blog/hacking-openai
