---
layout: post
title:  hooking fetch to read eventstream
tags: [fetch,eventstream]
---


# Using ChatGPT
when using ChatGPT, I would like to see the communications between the client and the server.  
If you open the Network panel in the DevTools, you can see a request to `https://chat.openai.com/backend-api/conversation`. But if you click the EventStream tab, you see nothing.  
Is something wrong? Is openai using some magic methods to hide the response?  

After some google stuff, we can get some answers in the Hacker News page.  
It's using eventsource. And there are some bug in Chrome, thus you can't see the response in the DevTools. And there is also a Chromium bugs page. There is still no progress.

# Can I get the response?
If DevTools doesn't show the response, can I show it myself?

## What is server-sent events?
Server-sent events is just like the websockets. But it's just a one-way connection. The client can't send events to the server, but the server can send events to the client.  

## try with chrome extension
In fact, chrome is getting stricter, it don't want you to read request body and response body now.
using `chrome.devtools.network`, you can read the response body.
`Note that request content is not provided as part of HAR for efficiency reasons. You may call request's getContent() method to retrieve content.`

```javascript
chrome.devtools.network.onRequestFinished.addListener((request) => {
  // Access request details and process them as needed
  console.log('Request URL:', request.request.url);
  console.log(request.response.content.text)

  request.getContent((content, encoding) => {
    console.log("Content: ", content);
    console.log("MIME type: ", encoding);
  });
});
```
I would like to say that, Mozilla has better developer documents than Chrome. But some APIs are not the same it the two browsers.  
if you try to log `request.response.content.text`, you will get nothing. You need to use `getContent` function.  

We can now get response of other requests. But we still can't get the response of `conversation`.  
Maybe it's still caused by the Chromium bug.

## Hooking fetch
We can't get the response with `chrome.devtools.network` API. Do we have other methods?  
If it's using `fetch` to send the request, can we hook the `fetch` function to log the response?
We asked chatgpt to write the function for us. And it worked really well. We just need to slightly modify the script.
```javascript
  fetch_ori = fetch
  fetch = fetchAndLog
  
  function fetchAndLog(url, options) {
  // Use fetch to make the network request
  return fetch_ori(url, options)
    .then(response => {
      // Clone the response so we can log it and return it
      const clonedResponse = response.clone();

      // Log the response (you can customize the logging as needed)
      console.log('Response from:', url);
      console.log('Status Code:', clonedResponse.status);
      console.log('Headers:', clonedResponse.headers);

      // Parse the response body as text or JSON, depending on the content type
      if (clonedResponse.headers.get('content-type').includes('application/json')) {
        return clonedResponse.json().then(data => {
          console.log('JSON Response:', data);
          return response; // Return the original response
        });
      } else {
        return clonedResponse.text().then(data => {
          console.log('Text Response:', data);
          return response; // Return the original response
        });
      }
    })
    .catch(error => {
      console.error('Fetch error:', error);
      throw error;
    });
}

```
Just paste the script in console and ask another question. Now you can see the response in the console. But we just lose the response effect.  
And a debug tip. Use `console.dir` to print the Object. You can see the Object instead of `[Object object]`.

## Now What?
If you see the response data, you would wonder why the response is so large. If fact, the response is nearly 800kB, while the result showing on page is only 3kB.   
Is openai potionally wasting its bandwidth?  
To makes the website have a timely response, it's sending what they get from the model. They are not using appending style. The just resend all the data again. So the response is so big. I think they can improve it by doing more at the client side. 

## If you want the fancy effect
This script can log in real time. But there is an error `DOMException: BodyStreamBuffer was aborted`.

```javascript
fetch_ori = fetch
fetch = fetchAndLog
  
 async function fetchAndLog(url, options) {
  // Use fetch to make the network request
  return fetch_ori(url, options)
    .then(async response => {
      // Clone the response so we can log it and return it
      const clonedResponse =  response.clone();
         logAnalyze(url, clonedResponse)
        return response
    })
    .catch(error => {
      console.error('Fetch error:', error);
      throw error;
    });
}

async function logAnalyze(url, clonedResponse){
     console.log('Response from:', url);
      console.log('Status Code:', clonedResponse.status);
      console.log('Headers:', clonedResponse.headers);
        const reader = clonedResponse.body.pipeThrough(new TextDecoderStream()).getReader();
function pump() {
          return reader.read().then(({ done, value }) => {
            // When no more data needs to be consumed, close the stream
            if (done) {
              return;
            }
            // Enqueue the next data chunk into our target stream
            console.log(value);
            return pump();
          })     .catch(error => {
        console.error('Fetch error:', error);
            // throw error;
         });;
        }
        pump()
    return 
}
```

https://news.ycombinator.com/item?id=35268197
https://github.com/Yaffle/EventSource/issues/79
https://bugs.chromium.org/p/chromium/issues/detail?id=1025893
https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
https://developer.chrome.com/docs/extensions/reference/devtools_network/#type-Request
http://www.softwareishard.com/blog/har-12-spec/#content
https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/devtools/network/onRequestFinished
