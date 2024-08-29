---
layout: post
title:  research on service worker
tags: [security,javascript]
---

最近看到SonarSource的一篇博客，roundcube webmail里面的xss漏洞。文中讲到可以利用service  worker的技巧来持久化xss。于是来研究一下。

## service worker
在mdn上我们可以看到，`Service workers essentially act as proxy servers that sit between web applications, the browser, and the network (when available). `
service worker可以加强离线体验，可以拦截请求，同时使用cache，来达到离线使用web应用的效果。

流程不多叙述，我们主要关注register，以及能够拦截哪些fetch。  
register(scriptURL, options)，包含scriptURL，options中包含scope。
通过实验，我们发现，scope是跟scriptURL的路径有关。而与包含register这段代码的脚本路径关系不大。

scriptURL最终会转换为绝对地址，我们以相对地址举例。
* 在子目录中  
scriptURL为/aa/sw.js，那么max  scope就是/aa/。当然 你的scope可以定义的更小。在这个scope下面，/aa/及其子目录如/aa/bb/，这里面 发起的请求是可以被拦截到的，比如/aa/bb/cc.html里面请求/top.js或者其他站点的js，这些请求都是可以获取到的。  
当你访问/index.html时，是不会被拦截到的。  
那么，在这种情况下，是否有可能扩大scope呢？也是有的，需要/aa/sw.js的返回Response Header里面包含Service-Worker-Allowed头，如Service-Worker-Allowed : /，这样就可以扩大范围。这种情况利用，还需要能够控制返回header，感觉 不太好利用。  
* 在根目录  
scriptURL为/sw.js，那么max scope就是/，那么你是可以拦截到任何请求了。

## 其他实验
* 是否能够替换javascript
在拦截范围内是可以替换的。但是不影响非拦截范围的请求。  
我们仍然以在子目录为例，/aa/sw.js  
比如访问/index.html，会请求/a.js文件。  
而/aa/test.html，里面也会请求/a.js文件，那么我在worker里面，修改/a.js的返回，是否能影响首页访问/index.html加载/a.js呢？  
并不能。 但是/aa/test.html访问/a.js时，是会被替换的。以及/aa/bb/cc.html访问/a.js时，也会被替换。这是因为 首页访问不在scope里面，所以service worker没有拦截，而另外两个在/aa/下面，是可以被拦截到的。

* 关于持久性
关闭tab之后，再重新打开，仍然是替换之后的结果，说明。确实拥有持久性。因为请求被service worker拦截，worker直接返回cache的结果。

## 结论
service worker确实可以持久化xss。不过和service worker.js这个文件的路径有关。如果能够控制worker的代码在根目录下，那影响就非常大。
感觉实际利用还是比较受限制的。

另外还有一点，就是sw.js的mime type还得是javascript的mime.

https://www.sonarsource.com/blog/government-emails-at-risk-critical-cross-site-scripting-vulnerability-in-roundcube-webmail/
https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers
https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerContainer/register
https://github.com/mdn/dom-examples/tree/main/service-worker/simple-service-worker
