---
layout: post
title:  如何只下载Github Repo的部分内容
tags: [github,tech]
---

最近KCon更新了2024年的PPT，但是是在整个Repo里面更新的。里面包含了之前的所有的PPT。

直接通过网络浏览里面的pdf文件，发现打开失败。点击下载，要么是下载不成功。要么是速度超级慢。

git clone试一试，超级慢，下着下着，就断了。

之后搜索看有没有只克隆部分目录的方法，结果Chatglm和doubao给的答案，pull 一个commit之后，checkout不成功，然后问它原因，也问不出来。
另外一个提到有sparse-checkout命令，尝试，结果git版本没有这个命令。升级git，然后又超级慢。

怎么办？有没有其他办法？
（备注：1.不想讲所有历史下载到.git中，2.不想讲其他不要的文件夹也下载到.git中）
## ugly way
这里介绍一个比较ugly的方法。
* 首先fork需要下载的大Repo
* 之后在Github Web界面上操作，删除不要的文件夹。（由于是一次性的工作，暂时没有自动化，就多点击几次）
* 由于我们没有办法fork这个fork的repo，所以我们需要进入设置，勾选“将repo设置为模板repo”
* 以模版repo生成一个新的new repo
* git clone 这个new repo

成功下载。

其他参考链接
https://www.baeldung.com/ops/git-clone-subdirectory
https://www.geeksforgeeks.org/how-to-clone-only-a-subdirectory-of-a-git-repository/
