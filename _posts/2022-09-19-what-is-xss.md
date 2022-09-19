---
layout: post
title:  什么是XSS
tags: [websec,xss]
---


# XSS是什么
XSS，Cross Site Scripting，跨站脚本攻击。一种在网站中注入脚本的攻击方式。  
攻击一般发生在前端，用户的浏览器中。  
为什么缩写不是CSS？大概是因为Cascading Style Sheets缩写是CSS。  
在最新的OWASP Top 10中，XSS已经合并到A03 Injection。

## XSS举例
如何直观了解XSS是什么？这里以在线靶场举例，
https://xss-game.appspot.com/level1/frame?query=aaa

query参数的值aaa被返回到页面中，如果我们将其改为xss payload，则触发xss。
https://xss-game.appspot.com/level1/frame?query=aaa%3Cscript%3Ealert(/xss/)%3C/script%3E

脚本被执行，同时我们查看页面代码，可以看的代码被返回到页面。

可以说，我们的payload被注入到页面中。由于HTML是文本流，标签和文本并没有差别，我们的payload中的标签也被解析并执行了。

## XSS的危害
在网站中注入脚本，有什么风险或危害呢？  
我们知道，网页上的交互动作，大部分需要Javascript来实现。Javascript能做的事情，就是XSS能做的事情。  
比如通过document.cookie读取浏览器中保存的Cookie，进而通过Cookie登录用户的账号；获取页面的敏感信息并发送请求将其传递出去；改变网页样式来进行钓鱼攻击；以当前浏览器为跳板，进一步刺探内网信息；以当前用户的权限发送请求，执行一些敏感操作；发动DOS攻击；制造XSS蠕虫等。在Nodejs的环境中，还可能执行命令。在浏览器特权页面，结合浏览器提供的接口，也可能执行系统命令。

## XSS分类
现在我们对XSS有了直观的了解，我们再学习XSS的分类。  
XSS根据Payload的触发方式，可以分为反射XSS、存储XSS和DOM XSS。这三种分别表示什么？
* 反射XSS，指XSS相关的payload在HTTP请求包中，随着HTTP响应包返回给浏览器前端，然后触发执行。比如前面的例子。
* 存储XSS，指XSS payload会事先被提交存储到网站，用户访问相应网页时，请求中没有携带xss payload，但因为事先已经被提交到网站，此时仍然会触发。比如在论坛中，有人在帖子中提交过xss payload，而网站存在xss漏洞，当你去访问对应帖子时，就会触发xss。
* DOM XSS，前面反射XSS的payload一般是比较直接地触发，不需要额外操作，而DOM XSS，是需要浏览器前端一些DOM操作才触发的。比如将参数不加处理，直接传入document.innerHTML等危险方法，进而导致payload被执行。

## XSS攻击
### Myspace XSS Worm（samy is my hero）
2005年，由于Myspace存在xss漏洞，用户Samy通过一段xss代码，修改被攻击用户的profile页面，添加xss代码，并留下“but most of all, samy is my hero”，同时将链接发送给好友。当好友浏览该profile页面时，同样的操作也会进行，这样就形成大量传播。20小时之内，超过100万用户中招。国内新浪微博也在2011年发生过xss蠕虫事件。社交类网站，特别需要注意xss的防范。
除了攻击广泛、速度快的蠕虫之外，还有一些普遍的攻击方式。
### 窃取Cookie
最容易利用的方式，通过获取document.cookie，再利用添加图片的方式将数据传递出去。不过由于现在httpOnly属性设置的较多，这种方式利用成功的场景在变少。
如果设置了httpOnly，是否还能获取到cookie呢？
在一些场景下还是可以的。Apache HTTP Server 2.2.x 到 2.2.21处理400错误时，会将cookie返回，包括httpOnly的cookie。因此可以利用这一点，获取httpOnly cookie。
类似地，还可以利用phpinfo页面来获取cookie，或者寻找网页中是否包含httpOnly cookie的信息。
如果获取不到cookie，还可以有其他利用方式。
比如现在不少网站，当用户未登录时可以看一点内容，当用户鼠标下滑一点，就会弹窗让用户登录，这种体验用户已经非常熟悉。如果我们利用xss代码，弹一个几乎一样的登录窗，用户会填写自己的用户名和密码吗？答案是，会。所以可以利用xss来做钓鱼攻击。
### 获取浏览器保存的密码
通过在浏览器中构造表单，等待浏览器自动填充之后，读取表单中的用户名和密码。
### 网页截图
利用HTML5 canvas API绘制网页，进行截图。
### 扫描
以当前浏览器为跳板，进行端口扫描。
### 执行命令
在Nodejs环境下，Electron Desktop Apps，如果开启了nodeIntegration，那么可以直接通过<script>require('child_process').exec('calc');</script> 直接执行命令。

### 平台型工具
#### XSS平台
对于前面提到的方法，现在有不少平台已经集成，可以自己搭建，组合使用，非常方便。
#### Beefproject
beef project集成了大量xss payload，可以以浏览器为跳板进一步利用。比如，如果浏览器版本较低，可能存在漏洞，那么可以通过xss触发浏览器漏洞，进一步获得更大的权限。

### 有趣的DOM XSS
为何要单独列出dom xss，因为通常来说，反射和存储xss，相对payload比较简单，dom xss由于跟前端实现有关，各有特点。
#### Mutation xss
```document.body.innerHTML="<form><math><mtext></form><form><mglyph><style></math><img src onerror=alert(1)>"```
请问读取document.body.innerHTML会返回什么？

我们可以看的document.body.innerHTML的值与我们的输入并不是一样的，即发生了mutation。利用这一特性，可以绕过一些前端xss防御，还会产生一些特殊的xss。
#### Client-Side Prototype Pollution
库是安全的吗？当我们使用一些特定版本的前端库时，存在原型污染。有些库，可以直接触发xss，有些库，结合代码，可以寻找到有用的利用点。
#### postMessage XSS
当postMessage没有判断targetOrigin，并且存在危险操作时，也可能触发dom xss。

## XSS防御
了解到XSS的原理之后，我们可以对症下药。
### WAF拦截
在纵深防御中，会前置WAF（web 应用防火墙）进行拦截。
### 输入处理
对于处理本身，非富文本处理，可以进行格式判断、长度限制、转义特殊字符的操作。
而对于富文本的处理，可以使用白名单的方式进行过滤。
而对于dom xss，一方面尽量不要使用危险的方法，如eval，innerHTML等。同时还可以尝试使用Trusted Types。
### 输出处理
使用Dompurify输出时进行净化。
### CSP限制
正确设置CSP，也可以部分限制xss攻击。


https://owasp.org/Top10/  
https://owasp.org/www-community/attacks/xss/  
https://www.acunetix.com/blog/web-security-zone/test-xss-skills-vulnerable-sites/  
https://en.wikipedia.org/wiki/Samy_(computer_worm)  
https://coolshell.cn/articles/4914.html  
https://www.cnblogs.com/52php/p/5658303.html  
https://research.securitum.com/mutation-xss-via-mathml-mutation-dompurify-2-0-17-bypass/  
https://github.com/BlackFan/client-side-prototype-pollution  
https://labs.detectify.com/2016/12/15/postmessage-xss-on-a-million-sites/  
https://web.dev/trusted-types/  
https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/xss-to-rce-electron-desktop-apps  
https://portswigger.net/web-security/cross-site-scripting  

