---
layout: post
title:  CSP Nonce closed the door with the window open
tags: [security,xss,csp]
---

CSP Nonce closed the door of XSS. But it leaves the window open. Just don't give up.

## Story
Last time, I write a blog about the end of XSS. I found a XSS, but the target has nonce and strict-dynamic csp rules.  
There's not an easy way to bypass it. So I thought it will be the end of XSS.  
After several days of trying, I bypassed it.  
CSP Nonce closed the door. But the window is still open. To be frank, the window is not widely open. So it is still the end of XSS.  

## The door
Normally, when we get a XSS, we can execute javascript.  
But with CSP enabled, we have less working payloads.  
If the CSP has white list domains, we can use it to load scripts and bypass the CSP.  
This is the easy way.  
But this time, the door is closed.  
The target has nonce and strict-dynamic enabled.  
With nonce, your script will have to have the same nonce. Usually, it'a random value. So your script will not match and exec.  
And with strict-dynamic, white list domains and unsafe-inline will be ignored. So we can't use white domains. And the on event will not work neither.

## The window
So the door closed.  
What can we do?  
The easy way is strict-dynamic. If a script can execute and it use `document.createElement("script")` to create and load a new script, the new script can execute.  
So we must find the sink whose script src can be controlled by us.  We tried this, but didn't find any.

One day, when I read a blog by slcyber, they also meet the CSP with strict-dynamic. And they use a gadget to bypass it.  
So I tried payload from gmsgadget. But the target does not have the exact library. So the payloads are not going to work.  
Here comes the window. Our last chance.  
The target has `unsafe-eval`. So if we can find a sink which evals our payloads, then we can bypass the csp.  
Easy?  
We searched `eval`, `setTimout`, `setInterval` and `new Function` .  
After days of debugging and understanding the minified javascript with AI, we didn't get a working sink.  
So I think we may be not able to exploit the xss.  
Several days later, I decided to give it another try.  
This time, I was checking another function of the target. With the CSP enabled, I still need to find a way to execute the script. Or the xss will not be useful.  
This time, with the help of AI, I dug deeper. After debugging, I get a working sink.  
The trigger point is the same with the sink I last checked. But it needs a new path. This is the reason I faild last time. Because I only tried the easy path. This time, I dug deeper and found the complex path.  
So don't give up. The door is closed by nonce, but the window of `unsafe-eval` may still be open. You just need some luck.

* https://portswigger.net/research/hunting-nonce-based-csp-bypasses-with-dynamic-analysis
* https://portswigger.net/research/bypassing-csp-via-dom-clobbering
* https://www.slcyber.io/research/a-jpeg-a-race-and-a-ghost-breaking-discourses-image-pipeline
* https://gmsgadget.com/
* https://portswigger.net/research/using-form-hijacking-to-bypass-csp

