---
layout: post
title: "细说Http Cookie与通行证安全"
description: ""
category: Security 
tags: [Web, Http, Cookie, Passport]
---
{% include JB/setup %}

对于web系统而言，由于HTTP协议无状态的特性，用户登录时需要服务端生成通行证返回给浏览器。浏览器保存该通行证并在接下来的请求中携带该通行证。通常来讲，web系统使用http cookie来保存和传输通行证。本文介绍http cookie的原理、特性、并分析用其保存通行证可能遇到的安全问题。

本文假设使用cookie的客户端是浏览器，虽然还有其它客户端也使用cookie，但普通用户使用更多的还是浏览器。

### 什么是Http Cookie？

一种Http状态管理机制，最早由Lou Montulli发明于1994年，最新的标准是2011年发布的[RFC 6265 - HTTP State Management Mechanism](https://tools.ietf.org/html/rfc6265)。因为保存信息到客户端的本质，也有系统用cookie来缓存信息，但这并不是Http Cookie的主要用途。

Cookie既指整个状态保存机制，也指用户保存状态的数据本身，由String类型的name和value以及若干属性组成。追根溯源，其名字来源于fortune cookie，一种内含写有幸运文字小纸条的饼干。

### Cookie保存在哪，如何生成又怎样传输？

Cookie由服务端生成，保存在浏览器。通过两个Http Header：Set-Cookie和Cookie进行传输。Set-Cookie Header用于服务端传输登录通行证等cookie键值对及属性给浏览器，浏览器收到并验证合法性后会做相应保存。Cookie Header用于浏览器传递cookie键值对给服务端，服务端依次来鉴别用户身份等状态信息。

RFC6265定义了Cookie的一些属性，包含Expires、Max-Age、Domain、Path、Secure、HttpOnly。也有一些还没定义到RFC，但已经实际应用的属性，比如SameSite。Cookie写时有属性读时无属性，属性仅在设置时即Set-Cookie Header中指定，浏览器依此来决定是否接受该cookie如何使用该cookie，而一旦浏览器决定了将cookie放到Cookie Header中传递到服务端，那么浏览器不会传递更多的的信息到服务端，只会传递键值对name和value。

![HttpCookie](http://zhaox.github.io/assets/images/HttpCookie.png)

很多浏览器会提供读Cookie和写Cookie的api给运行在其上Javascript脚本使用，通常的api是操作docment.cookie：

```
    document.cookie=“SID=31d4d; domain=example.com; path=/;”;
```

### 说说domain和path这两个属性

### Cookie的有效期过长导致通行证泄露

用户有使用公用电脑的情况。

### 明文传输的HTTP流量

### 利用XSS漏洞读取保存在cookie的通行证

### 利用XSS漏洞写cookie

利用服务端漏洞读cookie

使用户退登

更危险的情况

### 利用cookie大小和数量的限制来破坏安全

### 利用csrf漏洞偷用通行证

### 不安全的公共wifi

### 举个例子

### 总结






