---
layout: post
title:  javascript Object赋值(=)
tags: [javascript,tech]
---

javascript的引用赋值。

## 问题
最近写了一段代码，出错了。简化一下，是下面这样的逻辑：
```javascript
var a={}
a['a']=1
for(var i=0; i<5;i++){
	var b=a;
	b['b']=i;
	console.log(b)
}
console.log(a)
```
请问最终输出的a是什么？
```javascript
{a: 1, b: 4}
```
console下执行一下，输出结果如上。

为啥a的内容被改了？
## 原因
查阅网络上的资料，是因此javascript的赋值分为值赋值和引用赋值。
对于Object和数组，采用的是引用赋值，即b和a指向同一个地址，对b的修改，也会导致a的修改。
这就是为什么我们在循环里面操作b的值，导致a的值也被修改了。

## 其他
在看文档时，发现const也是可以被修改的。
```
The const declaration declares block-scoped local variables. The value of a constant can't be changed through reassignment using the assignment operator, but if a constant is an object, its properties can be added, updated, or removed.
```
之前以为const是常量，不能修改。但其实如稳定所说，如果是object，属性还是可以被修改的。
```javascript
const a = {'a':1};
console.log(a);
a['b']=2
console.log(a)
var b=a
b['c']='c'
console.log(b,a)

> Object { a: 1 }
> Object { a: 1, b: 2 }
> Object { a: 1, b: 2, c: "c" } Object { a: 1, b: 2, c: "c" }
```


https://www.cnblogs.com/cench/p/6019453.html
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Assignment
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/var

https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const

