---
layout: post
title:  emudhra 中间证书弱密钥
tags: [security,ssl,ca]
---

emudhra中间证书使用了弱密钥。其中一个证书密钥是4。

## 中间证书
什么是中间证书？  
这个需要说明背景。通常来讲，比如你使用浏览器上网，访问https的网站时，网站会返回它的证书给你做校验。  
那么浏览器如何做校验呢？  
使用了非对称加密的方式来做校验。简单来说，useragent校验证书是来自一个useragent信任的机构颁发的证书，否则不予信任。  
那么如果知道哪些机构可以信任呢？即证书的上级是可以信任的呢？最简单方法，就是你的useragent里面包含了一个可以信任的证书，而网站证书是来自你的useragent里面的证书签发的。  
这些可以信任的证书是随着你下载或者更新useragent的时候，一起更新的。这里有一个风险，就是假设这些证书泄漏了，那么影响范围比较广，更新起来也麻烦。  这个时候，引入一个中间证书，如网站证书-中间证书-根证书，这样的结构，讲根证书严格保护起来，平常做证书签发使用的是中间证书。这样，假设出问题，更容易出的是中间证书这环，如果使用很多的中间证书，那么影响范围也会相对小一点，只需要处理这张中间证书签发的相关证书即可。  

## 非对称加密
那么，如果做验证呢？  
比如中间证书使用自己的私钥来签名一个网站的证书，那么useragent可以使用中间证书的公钥来验证是否确实是这个中间证书签名的。常见的非对称加密有RSA、椭圆曲线。  
而这里的要点就是，这个私钥是保密的。通常情况下，公钥和私钥都很大，并且通过公钥难以推断出私钥。  

## emudhra
那么emudhra这次是什么问题？emudhra的几个中间证书，私钥都是非常弱的，最弱的就是4。  
我们使用ai来验证一下。  
首先我们从问题披露网站下载proof.zip，使用文本工具查看其中的一个pub.pem。直接让ai阶段这个pem。
可以得到序列号 A4AEE678F23362B4C0E460B772F214，签名算法	ecdsa-with-SHA384，签发者 Issuer	C=IN, O=eMudhra Technologies Limited, CN=emSign Root TLS CA - G3，持有者 Subject	C=NL, O=eMudhra BV, CN=EM DV TLS CA - G3B，公钥	id-ecPublicKey / secp384r1 (P-384)  
ai表示，这个序列号就是这次安全事件中的证书之一。  
接着我们让ai来破解这个证书的私钥，ai生成了一段暴力破解的代码，运行之后，得到k为156。  

```bash
python3 - <<'PY'
import base64
# P-384 (secp384r1) 曲线参数
p  = 2**384 - 2**128 - 2**96 + 2**32 - 1
a  = -3 % p
b  = int('b3312fa7e23ee7e4988e056be3f82d19181d9c6efe8141120314088f5013875ac656398d8a2ed19d2a85c8edd3ec2aef',16)
Gx = int('aa87ca22be8b05378eb1c71ef320ad746e1d3b628ba79b9859f741e082542a385502f25dbf55296c3a545e3872760ab7',16)
Gy = int('3617de4a96262c6f5d9e98bf9292dc29f8f41dbd289a147ce9da3113b5f0b8c00a60b1ce1d7e819d7a431d7c90ea0e5f',16)

raw = open('em_clean.pem').read()
b64 = ''.join(l for l in raw.splitlines() if l and 'CERTIFICATE' not in l)
b64 = b64[:-1]                      # 去掉末尾疑似多余的字符
der = base64.b64decode(b64)

# 从 DER 提取未压缩公钥点：BIT STRING 位于偏移248，未用位1字节+97字节点
pt = der[251:251+97]
print("提取公钥点长度:", len(pt), "格式字节:", hex(pt[0]))
x = int.from_bytes(pt[1:49],'big'); y = int.from_bytes(pt[49:97],'big')
on_curve = (y*y - (x*x*x + a*x + b)) % p == 0
print("公钥点是否在 P-384 曲线上:", on_curve)
if not on_curve:
    print("-> 公钥点被截断，无法继续验证"); raise SystemExit

def inv(z): return pow(z % p, -1, p)
def add(P,Q):
    if P is None: return Q
    if Q is None: return P
    X1,Y1=P; X2,Y2=Q
    if X1==X2 and (Y1+Y2)%p==0: return None
    m = (3*X1*X1+a)*inv(2*Y1)%p if P==Q else (Y2-Y1)*inv((X2-X1)%p)%p
    X3=(m*m-X1-X2)%p; Y3=(m*(X1-X3)-Y1)%p
    return (X3,Y3)
def mul(k,P):
    R=None
    while k:
        if k&1: R=add(R,P)
        P=add(P,P); k>>=1
    return R

Q=(x,y); G=(Gx,Gy)
print("开始枚举 k=1..255（弱密钥恢复演示，任何人都能做）...")
found=None
for k in range(1,256):
    if mul(k,G)==Q:
        found=k; break
print("验证结论:", f"私钥 = {found}（小于256，可被从公钥直接反推 → 确认为弱密钥）" if found else "1..255 内未找到（私钥≥256 或 点不完整）")
PY
```
ai表示，`你贴的这张 EM DV TLS CA - G3B 证书，其对应的私钥就是 k = 156——一个小于 256 的普通整数。任何人只要拿到这张证书（里面只有公钥），花不到一秒钟枚举 1–255 就能把私钥算出来。`  
这里本来的难点是，证书的私钥非常大，这样通过暴力破解，需要很长很长的时间，因此，暴力破解基本不可行。但这次的问题是私钥其实是非常小的数字。  
那么问题就是，你拥有了这张中间证书的私钥，你也可以用它来签署其他证书。  

如果你打开浏览器里面的证书页面，你是可以找到emSign Root CA - G3的证书的，如果这张中间证书仍然有效的话，你使用它签发的证书，最终通过证书链，就可以达到root ca，进而被浏览器信任。  
不过，目前几大浏览器已经通过自己的方式标记了这几个证书。  
但是，对于不使用这些证书标记的useragent，则可能需要很久才能获得更新。


https://emudhra.we-have-a-problem.com/
