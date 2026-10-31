---
layout: post
title:  XSS CSP and AI
tags: [security,xss,csp,ai]
---

One week. 500 points of AI tokens. XSS. Another CSP Bypass. And Don't give up.
## XSS
Last time, I found a XSS and a CSP bypass on the target. I thought it would be the end of XSS. So I rested for a while.  
Several days later, I thought I would give it another try.  
So first, I will have to find a CSP bypass. Because finding a XSS is not that hard. If we don't have a CSP bypass, our XSS will not get a bounty.  
After several days of trying to review the CSP sink points, I didn't get luck.  
So I try to use AI to help me understand the javascript code and see if AI can find vulnerabilities.  
One day AI spotted a XSS. And I tried it. Yes, it's positive.  
Now comes the hard part.
## CSP
Another days of understanding the code, AI spotted a potential CSP bypass.  
After trying it, no alert fired.  
Trying to debug the code, it seems that our payload didn't execute.  
And after several conversations with AI, AI thought that this bypass may not work.  
In fact, the bypass technique is very difficult to understand. On the first sight, I didn't understand why the code would run.(AI is really good. The code is minified. I can't spot the bypass with my small screen. The height of my screen is the length of my context. :) )  
More conversations with AI. And sometimes AI would say that it needs to modify the previous answers. (Well, I try to understand AI's previous anwsers, and I almost understood it. Then AI say that the previous answer is not correct. Bad AI.)  
So more tokens. More debugging. And all the leads trace back to a config variable. And it's hardcoded. So the payload would not fire.  
Trying more methods. Searching more sinks. Still no bypass. And I think I need to quit.
## AI
I think I will give it another try. If I still can't bypass it. I will just skip this bypass and try to find another bypass.  
So I think I need to understand more about the technique of the bypass. So I asked AI to generate a small poc of this technique to understand why this works and what is essential.  
After testing another sink point, it seems promising. And AI spotted a sink point of this technique. First try. Failed. Asked AI more.  Another try. Succeed. Thrilling. Exciting.
## Conclusion
So I submitted the vulnerability. And checked that I understand all the aspects of the XSS and the CSP bypass.  
Finding XSS is easy. Bypass CSP is hard. AI can find XSS and also CSP Bypass. Don't give up. And you should use AI to help you understand the code.

