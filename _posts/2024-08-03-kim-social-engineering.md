---
layout: post
title:  朝鲜黑客：找工作吗？会成为肉鸡的那种
tags: [社工,hack]
---

前段时间，KnowBe4招了个朝鲜黑客。结果一搜，发现朝鲜黑客不但伪装成美国人去应聘，他们还伪装成老板来招人。当然，他们不是真的想招你，真正的目的是想黑了你的电脑。

让你自愿运行恶意程序，还有比这更“好”的事吗？

## 前期准备 
首先是进行一些环境和账号的准备，比如创建Github账号，之后会创建仓库，刷一点活跃度，做一些提交，让自己的账号看起来是一个真实的账号。

之后会在里面藏入恶意代码。
![gh](/static/img/github.png)

还需要准备一些邮箱，社交平台账号（LinkedIn，TG，WA等）。
## 开始行动
在linkedin（国外职场社交平台）上建立账号，比如伪装成大公司的招聘人员，例如下面就是伪装成脸书公司的招聘人员。  
![fake linkedin](/static/img/linkedin.png)  
(pic from eset)  
通过平台广告，电子邮件或者在社交平台直接联系潜在的受害者。

当有人上钩之后：
1. 直接聊天  
比如直接通过聊天的方式，想要验证对方的能力，让对方完成一些编程题，然后将在Github上准备的文件发给对方。一旦对方运行，肉鸡成功上线。
2. 面试  
有些时候，他们还会“贴心”地给你一个带有后门的视频软件，这样你就可以通过这个软件进行面试了。当然，他们也不是随便找一个软件，而是找一个比较有名的软件，通过修改这个软件的一些文件来加入后门，所以，看起来仍然是正常的软件。一旦运行，点击特定的菜单，肉鸡成功上线。
3. 面试做题  
参考一些大公司的流程，通过发布软件工程师的招聘信息，之后作为面试官，对应聘对象进行面试。  
面试的过程中，会要求面试者完成一些开发任务，需要使用Github上面的代码。而这些代码中，藏有一些恶意代码。面试者一旦执行，电脑就会被控制。

## 为什么会有人中招
因为代码里面会做混淆。还会使用非常多的空格，将代码藏在很靠后的地方，如果你不仔细，可能也不会发现。  
![空行](/static/img/sespace.png)  
另外各种库文件，你也没有办法一个一个地去看。  
有些时候，他们还会“贴心”地给你一个带有后门的视频软件，这样你就可以通过这个软件进行面试了。当然，他们也不是随便找一个软件，而是招一个比较有名的软件，通过修改这个软件的一些文件来加入后门，所以，看起来仍然是正常的软件。  
攻击者会使用看起来合法的公司，同时流程也和真实流程类似。因为有不少公司是需要面试的过程中进行现场编程的，所以流程上是可以接受的。  
