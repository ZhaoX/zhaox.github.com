---
layout: post
title: "白话Https"
description: ""
category: Security
tags: [Https, Cryptography]
---
{% include JB/setup %}

本文试图以通俗易通的方式介绍Https的工作原理，不纠结具体的术语，不考证严格的流程。我相信弄懂了原理之后，到了具体操作和实现的时候，方向就不会错，然后条条大路通罗马。阅读文本需要提前大致了解对称加密、非对称加密、信息认证等密码学知识。如果你不太了解，可以阅读Erlang发明人Joe Armstrong最近写的[Cryptography Tutorial](https://github.com/joearms/crypto_tutorial)。大牛出品，通俗易懂，强力推荐。

###Https涉及到的主体

1. 客户端。通常是浏览器(Chrome、IE、FireFox等)，也可以自己编写的各种语言的客户端程序。
2. 服务端。一般指支持Https的网站，比如github、支付宝。
3. CA(Certificate Authorities)机构。Https证书签发和管理机构，比如Symantec、Comodo、GoDaddy、GlobalSign。

下图里我画出了这几个角色：
![Http Role](http://zhaox.github.io/assets/images/HttpsRole.png)

###Https存在的动机

1. 认证正在访问的网站。什么叫认证网站？比如你正在访问支付宝，怎样确定你正在访问的是阿里巴巴提供的支付宝而不是假冒伪劣的钓鱼网站呢？
2. 保证所传输数据的私密性和完整性。众所周知，Http是明文传输的，所以处在同一网络中的其它用户可以



###总结

1. 说是讨论Https，事实上Https就是Http跑在SSl或者TLS上，所以本文讨论的原理和流程其实是SSL和TLS的流程，对于其它使用SSL或者TLS的应用层协议，本文内容一样有效。
2. 本文只讨论了客户端验证服务端，服务端也可以给客户端颁发证书并验证客户端，做双向验证，但应用没有那么广泛，原理类似。
