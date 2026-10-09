---
layout: post
title:  The end of XSS
tags: [security,xss]
---

AI found a XSS by reviewing the javascript code. But CSP blocked it. And this may be the end of XSS.

## AI
I am trying to find a XSS on a target. It has rich text.  
When I try to understand the logic of the new functions, I tried the AI.  
Because mordern javascript has many modules and the function calls are here and there, and the code is also minified, the code is not easy to read. So I tried to use AI to analyze the source code.  
I didn't expect that there were any vulnerabilities in the code. I just want to understand the code and how the user input is turned into the output.  
In fact, the AI is really good at understanding the source code. It returns the logic of the code. The logic is complex.  
So I asked if there were any vulnerabilities in the source code. And several minutes later, it told me there is a XSS in the source code. After a test, the AI is right. A special input can be turned into a html tag. So, we can input html tag now.

## CSP
I know this site has CSP. So I tried a payload which can bypass the CSP.  
But this time, it won't work.  
So I checked the response header. It turns out that they used a stricter CSP this time.  
`'nonce-{random string}' 'strict-dynamic'`. They used `nonce` and `strict-dynamic` now. Last time, I didn't see this in their CSP.  
After trying to bypass the CSP for serveral days and failed, I would say that this is the end of XSS.  
Although there may be ways to bypass the CSP, the roads are narrowed.  
With `nonce` and `strict-dynamic`, the white domains are ignored. So I can't bypass it with a white domain now.
According to invicti blog,`In CSP Level 3 browsers, 'strict-dynamic' causes host sources, scheme sources, 'self', and 'unsafe-inline' to be ignored for script loading when nonce or hash trust is present.` So we can not use inline scripts. And the target's CSP also does not have `unsafe-inline` in it.  

One way to bypass the CSP is to combine another sink point which can use your input. But it's not easy to find it in the complex source codes. I may try it later.

## conclusion
If you want to find a XSS on a target, you should first check their CSP and judge where you can bypass it. If it's a very strict policy, even if you find a XSS, you can not exploit it. And a xss which can not be explited will not get a good bug bounty.  
So I think this may be the end of Exploitable XSS.

* https://www.invicti.com/blog/web-security/negative-impact-incorrect-csp-implementations
* https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CSP
