---
layout: post
title:  GraphQl安全之DVGA学习
subtitle: GraphQl安全
tags: [graphql,websec]
---

## 如何识别
### 接口
比较明显的是接口`/graphql`，还有一些其他的接口名，可用字典扫描。
### 类型
一般是json格式的type，`Content-Type: application/json`
### 数据
带有query的json格式数据
`{"query": "query getPastes {\n        pastes(public:true) {\r\ntitle\r\n          id\n          title\n          content\n          ipAddr\n          userAgent\r\n\n          owner {\n            name\n          }\n          }\n        }"}`

## Introspection
运气好的话，可以通过Introspection来获取表信息。有相应的burp插件可以帮你完成，但是经常会被waf拦截。
```json
{"query": "query IntrospectionQuery{__schema{queryType{name}mutationType{name}subscriptionType{name}types{...FullType}directives{name description locations args{...InputValue}}}}fragment FullType on __Type{kind name description fields(includeDeprecated:true){name description args{...InputValue}type{...TypeRef}isDeprecated deprecationReason}inputFields{...InputValue}interfaces{...TypeRef}enumValues(includeDeprecated:true){name description isDeprecated deprecationReason}possibleTypes{...TypeRef}}fragment InputValue on __InputValue{name description type{...TypeRef}defaultValue}fragment TypeRef on __Type{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name}}}}}}}}"}
```
如果运气不好，那就只能拿字典去爆破了。

## 查询
比如正常访问，
```
query getPastes {
        pastes(public:true) {
          id
          title
          content
          ipAddr
          userAgent
          owner {
            name
          }
          }
        }
```
通过前面的Introspection，我们看到paste
```
query {
	paste(pId:"code") {
		owner {
			id
		}
		burn
		Owner {
			id
		}
		userAgent
		pId
		title
		ownerId
		content
		ipAddr
		public
		id
	}
}
```
那么我们就可以直接构造查询paste
```
query getPastes {
        pastes(public:true) {
          id
          title
          content
          ipAddr
          userAgent
          owner {
            name
          }
          }
paste(pId:"2") {
		owner {
			id
		}
		burn
		Owner {
			id
		}
		userAgent
		pId
		title
		ownerId
		content
		ipAddr
		public
		id
	}
        }
```
查询非常灵活。只要你知道字段的名字。

## 想法
由于DVGA本身提供了解答，解答过程就不写了。  
实际碰到过，但是不能Introspection，并且有waf。  
更多的是未授权的访问，或者基于接口的漏洞。但前提是运气（字典）足够好，能够找到对应的字段。  
有点像mysql information_schema和access一样。一个可以读到元数据，一个靠字典。  
工具也有不少，实际结合工具操作更方便。  
json可以多条查询，参数和查询可以分开写，有些字段可有可无。实际成功利用了再来更新。


## 引用
1. [Damn Vulnerable GraphQL Application](https://github.com/dolevf/Damn-Vulnerable-GraphQL-Application)
2. [Introspection](https://graphql.org/learn/introspection/)
3. [How to exploit GraphQL endpoint: introspection, query, mutations & tools](https://blog.yeswehack.com/yeswerhackers/how-exploit-graphql-endpoint-bug-bounty/)
