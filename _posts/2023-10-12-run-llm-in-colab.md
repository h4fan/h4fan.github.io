---
layout: post
title:  我想运行大模型，但是没有A100
tags: [llm,colab]
---

# 如何免费运行大模型？

上次用CPU运行ChatGLM2之后，CPU风扇呼呼地转，有没有免费的GPU资源来运行？  

答案是，有的，来看一看Colab。  

# What is Colab?
```
Colab, or "Colaboratory", allows you to write and execute Python in your browser, with
Zero configuration required
Access to GPUs free of charge
Easy sharing
Whether you're a student, a data scientist or an AI researcher, Colab can make your work easier. Watch Introduction to Colab to learn more, or just get started below!
```

Accessto GPUs free of charge，显存15GB的GPU，不要钱，心动了吗？

# 如何使用

## 第一步：注册Colab账号

注册账号时，一直提示我名字重复，差点我就放弃了。

## 第二步：开启Drive

之后将模型下到Drive中，这样不用每次重新下载。

## 第三步：下载模型到Colab

参考https://github.com/lewangdev/chatglm2-6b-colab

将其中的地址改为`/content/drive/MyDrive/``

```bash
%cd /content/drive/MyDrive/
!sed -i "s/THUDM\/chatglm2-6b/\/content\/drive\/MyDrive\/ChatGLM2-6B\/models/g" web_demo.py
```
# Debug

运行之后，在web端总是显示不正常，查看网络流量，发现返回数据包里面已经有chatglm2的返回结果。

将web_demo.py中的第27行注释掉

```
# gr.Chatbot.postprocess = postprocess
```
最终成功运行，运行流畅，终于不用听cpu风扇狂转了。
