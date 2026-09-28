---
layout: post
title:  intigriti challenge 0826 2026 thoughts
tags: [security,ctf]
---

If you don't know the tricks, you won't find the answers.

## challenge
After finishing the 0926 challenge, I will give 0826 a try.  But in the end, I didn't find the answer. So this is not a writeup. It's just my thoughts about ctf.  

After open the page, we can see that this challenge also needs a `INTIGRITI{.*}` Flag. So I thought it's also a server side challenge.  

First, we check the source code. Nothing special. We can see a `static.mp4` url. And a `load` request, but the header is commented. And a `report` api. Nothing more.  

So I thought it's a server side challenge and I try to test the `load` api. After different payloads, it seems that the param in the path can only be numbers. Other payloads would give you a `403` response. And I also tried to append the `X-Channel-Id` header, but it seems the backend is not using it. I thougt maybe the backend checks the first param in the path, but it reads the channel in the header. But the results do not confirm this.  

Then I try the `report` api. Any payloads would only get the same response. I thought maybe the `id` in the response is useful and I tried to use it as the channel Id.  
After some other payloads, I didn't get what the author is checking. So I try to get some hint.

## Hint
I go to the writeups page. https://bugology.intigriti.io/intigriti-monthly-challenges/0826 . And I see the category is `Blind XSS, CSP Bypass, Mystery`.  
Oh. I am in the wrong way. I thought it's server side vulns like sqli or lfi. But it's xss and blind.  
So I tried a xss payload `<img src="//my-server/img" onerror=alert(1)>`. And I get a request log on my server.So the tag is inserted into the page. Maybe a XSS.  
So I think it's blind. Then I need to read the source code or the cookie. I tried many payloads and I didn't get the request from the backend.
```
referer : http://web/
user-agent : Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/129.0.6668.29 Safari/537.36
```
The image request revealed something. So I think maybe it's a chrome csp bypass. So I asked an AI to give me `HeadlessChrome/129.0.6668.29 csp bypass`. But I didn't get good results.  
So I tried some common bypass payloads and didn't get a request from the backend.  
And from here, I decided to see some writeups. Or it will be a waste of time. Because if you don't know the tricks, you won't get the answer.

## writeup
The writeup page is https://bugology.intigriti.io/intigriti-monthly-challenges/0826 .
The first writeup is from firecrackers. And firecracker exploited a trick in the HeadlessChrome/129.0.6668.29 to bypass the CSP. I would say this is difficult. If you don't play ctfs regularly, you wouldn't think of it.  
The second is etherwa. From etherwa's report, we see an easy one. The report page has CSP. But we can use an endpoint on the same domain to bypass it. There is an `jsonp` url. You can use to reflect the payload back. Then you can use it to bypass the csp and execute the javascript payloads. And you need to use the `load` api to get the channelid `11`, then you can get the mp4, which has the flag.  
I think cerberes's writeup has more details on what cerberes tried and how cerberes  got the conclusion. It's more helpful.  
So my thought is that, if you don't play the ctf regularly, you would not find the right way.
