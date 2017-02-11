---
layout: post
title: "细说Http Cookie与通行证安全"
description: ""
category: Security 
tags: [Web, Http, Cookie, Passport]
---
{% include JB/setup %}

对于web系统而言，由于HTTP协议无状态的特性，用户登录时需要服务端生成通行证返回给浏览器。浏览器保存该通行证并在接下来的请求中携带该通行证。通常来讲，web系统使用http cookie来保存和传输通行证。本文介绍http cookie的原理、特性、并分析用其保存通行证可能遇到的安全问题。

### 什么是Http Cookie？

一种Http状态管理机制，最早由Lou Montulli发明于1994年，最新的标准是2011年发布的[RFC 6265 - HTTP State Management Mechanism](https://tools.ietf.org/html/rfc6265)。

因为保存信息到客户端的本质，也有系统用cookie来缓存信息，但这并不是Http Cookie的主要用途。

Cookie即指整个状态保存机制，也指用户保存状态的数据本身。追根溯源，其名字来源于fortune cookie，一种内含写有幸运文字小纸条的饼干。

### Cookie保存在哪，如何生成又怎样传输？

### Cookie的基本特性

### Cookie的属性


