---
layout: post
title:  保存response body到文件-burp插件
tags: [burp,extension]
---
 

# 问题
最近在测试中，经常碰到文件下载的数据包，修改参数之后，返回的是文件，不好查看内容是否改变。  
直接save to file时，会包含response headers，仍需要再次编辑删除header头才能正常打开文件。

# 插件
结合官方样例和AI，快速地写出代码。  
目前在editor框里面，点击右键，选择extension，选择保存的位置，即可将内容保存出去。之后可以直接打开。

# 下载地址
https://github.com/h4fan/savebodytofile



