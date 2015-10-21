---
layout: post
title: "MAC与HMAC的介绍及其在S3和Azure中的应用"
description: ""
category: Algorithm 
tags: [Algorithm, Security, Erlang]
---
{% include JB/setup %}

#MAC 

在密码学中，（消息认证码）Message Authentication Code是用来认证消息的比较短的信息。换言之，MAC用来保证消息的数据完整性和消息的数据源认证。

MAC由消息本身和一个密钥经过一系列计算产生，用于生成MAC的算法，称为MAC算法。MAC算法应能满足如下几个条件:

1. 在仅有消息本身没有密钥的情况下，无法得到该消息的MAC；
2. 同一个消息在使用不同密钥的情况下，生成的MAC应当无关联；
3. 在已有一系列消息以及其MAC时，给定一个新的消息，无法得到该消息的MAC。

下图摘自维基百科，可以很好的描述MAC的使用原理：

![MAC](https://upload.wikimedia.org/wikipedia/commons/thumb/0/08/MAC.svg/661px-MAC.svg.png)

#HMAC

HMAC是MAC算法中的一种，其基于加密HASH算法实现。任何加密HASH, 比如MD5、SHA256等，都可以用来实现HMAC算法，其相应的算法称为HMAC-MD5、HMAC-SHA256等。

以下伪代码，描述了HMAC算法的计算过程：

```
function hmac (key, message)
    if (length(key) > blocksize) then
        key = hash(key) // keys longer than blocksize are shortened
    end if
    if (length(key) < blocksize) then
        key = key ∥ [0x00 * (blocksize - length(key))] // keys shorter than blocksize are zero-padded (where ∥ is concatenation)
    end if
   
    o_key_pad = [0x5c * blocksize] ⊕ key // Where blocksize is that of the underlying hash function
    i_key_pad = [0x36 * blocksize] ⊕ key // Where ⊕ is exclusive or (XOR)
   
    return hash(o_key_pad ∥ hash(i_key_pad ∥ message)) // Where ∥ is concatenation
end function
```

我实现了一个Erlang版本的HMAC-SHA256，可以在[我的github](https://github.com/ZhaoX/hmac_sha256)看到。

#MAC在S3和Azure Storage中的应用

MAC是对称加密算法，S3和Azure Storage都使用其来做请求认证。

在S3中，使用的是HMAC-SHA1：
```
Authorization = "AWS" + " " + AccessKeyId + ":" + Signature;
Signature = 
    Base64(HMAC-SHA1(AccessKeySecret, UTF8-Encoding(StringToSign)));
StringToSign = HTTP-Verb + "\n" +
    Content-MD5 + "\n" +
    Content-Type + "\n" +
    Date + "\n" +
    CanonicalizedAmzHeaders +
    CanonicalizedResource;
```

在Azure Storage中，[使用的是HMAC-SHA256](https://msdn.microsoft.com/library/azure/dd179428.aspx).
