---
layout: post
title:  interesting tagName xss payload
tags: [security,xss]
---

Gareth Heyes发了一个有意思的xss payload，`<alert(1) onfocus="attributes[0].value=localName,new onfocus" autofocus tabindex=1>`。试了一下，确实可以执行。

# 解读
首先运行了一下，查看了attributes和localName，attributes[0]就是当前的onfocus，而localName就是当前tagName的小写。  
运行之后，如果我们查看网页代码，发现onfocus已经被替换为alert(1)了，所有，相当于localName就是tagName，即alert(1)，被赋值给onfocus了，之后再次运行了onfocus，payload成功运行。  
确实是一个很有意思的payload。

# 实验
在原代码的基础上，我们做了一些实验。  
代码需要用到on event，向onfocus这种，可以自动focus，自动触发，无交互。  
同时，我们也可以使用onclick、onmouseover这类tag，不过需要有交互才能触发。  
同时tagName甚至可以输入各种特殊符号。
```
<console.log(document.domain);a=1;console.log("&") onmouseover="attributes[0].value=localName,new onmouseover" style="a:1"><div width=1000 height=1000>aaa</div>
```
比如这段代码，也是可以成功执行的，不过我们使用的是onmouseover，需要鼠标滑过。

# 总结
可以看出，javascript真是非常神奇的语言。  
目前看，这段代码需要用到on event，从实践中看，很多网站都过滤了on事件，倒是有可能在过滤非常严格的情况下使用，不过那种估计是ctf场景了。