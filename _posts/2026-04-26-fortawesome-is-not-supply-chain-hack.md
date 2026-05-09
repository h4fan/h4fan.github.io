---
layout: post
title:  fortawesome并不是供应链安全事件
tags: [security,tech]
---

fortawesome 并不是供应链安全事件。

## 新博客
作为一个创建博客爱好者，我又创建了一个新的博客，已经不知道这是第多少个博客了，反正肯定不是最后一个。

对，就是你正在看的这个博客。刚刚选好模板创建成功，顺手打开控制台，看了一下网络请求。

映入眼帘的就是一个`https://cdn.jsdelivr.net/npm/@fortawesome/fontawesome-free@latest/css/all.min.css`，仔细再看一眼，fortawesome,fontawesome。

心想，sh * t,供应链黑客？被黑了？

## dig
### 直接访问
带着一丝怀疑，直接访问了一下网址，确实是一个css，并且文件里面显示的fontawesome，很有供应链黑客的味道。
```
/*!
 * Font Awesome Free 7.2.0 by @fontawesome - https://fontawesome.com
 * License - https://fontawesome.com/license/free (Icons: CC BY 4.0, Fonts: SIL OFL 1.1, Code: MIT License)
 * Copyright 2026 Fonticons, Inc.
 */
```
接着访问`https://www.jsdelivr.com/package/npm/@fortawesome/fontawesome-free`，发现这个页面显示每个月访问几亿次，这么大的流量，不像是个假的。
访问`https://www.jsdelivr.com/?query=author%3A%20FortAwesome`，发现这个账号发了很多，并且下载量都非常大。

### 问ai
带着疑问，问了一下deepseek，这个fortawesome是否是malicious的，deepseek说`The name might sound unusual, but @fortawesome/fontawesome-free is the official, legitimate, and widely trusted package for including Font Awesome icons in your projects. The name is simply the GitHub organization (FortAwesome) that maintains the library.`，看着不正常，但其实是官方的。

### github
访问了一下github`https://github.com/FortAwesome`，确实是叫fortawesome，并且font-awesome库也在这个账户下，并且有70几k的star。

### 搜索
打开搜索引擎搜了一下fontawesome和fortawesome两个网站都有，其中有个reddit链接`/r/learnjavascript/comments/14cs62l/whats_up_with_fontawesome/`表达了同样的疑问。

有人指出，这个公司原来叫fonticons，（这个确实是与`fontawesome.com`网站下的的`© Fonticons, Inc.`符合的，）只是改名字叫fortawesome了，并给出了一个链接，`https://articles.fortawesome.com/fonticons-is-now-fort-awesome-fb19fa9dda46`。访问之后，确实是改名的公告。`Fonticons is now Fort Awesome`。发布时间是Jan 12, 2016，十年前。

所以，fontawesome确实是在fortawesome下面的。这并不是供应链安全事件。不过，在这个供应链安全事件频发的年代，这个名字确实是有点吓人的。
