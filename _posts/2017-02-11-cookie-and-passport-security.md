---
layout: post
title: "Cookie与Passport安全"
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

### 说说Domain和Path这两个属性

Domain、Path、Name三者唯一确定一个cookie。其它属性仅用作读写时的权限控制，不作为cookie标识。在浏览器在使用cookie时：
- 不区分Http/Https
- 不区分端口号

这相比浏览器同源策略更加宽松，带来很多安全问题。

Domain是向上通配的：

- 写入（Set-Cookie）访问www.example.com

```javascript
    Set-Cookie: sid1=a; domain=example.com; path=/;            接受
    Set-Cookie: sid2=b; domain=www.example.com; path=/;        接受
    Set-Cookie: sid3=c; domain=pay.example.com; path=/;        拒绝
```

- 读取（Cookie）

```javascript
    访问pay.example.com
		Cookie: sid1=a
		
    访问www.example.com
	    Cookie: sid1=a; sid2=b;
```


很多系统写cookie，习惯在域名前加一个点，写成.example.com，以为这么写才是通配的，其实这个点是多余的，没有必要的。

Path是向下通配的：

- 写入（Set-Cookie）

```javascript

    Set‐Cookie: sid1=a; domain=example.com; path=/;
    Set‐Cookie: sid2=b; domain=example.com; path=/test/;
```

- 读取（Cookie）

```javascript

    访问http://example.com/
	    Cookie: sid1=a;
	
	访问http://example.com/test/
	    Cookie: sid1=a; sid2=b
```

	
### Cookie的有效期过长导致通行证泄露

这两个属性指定cookie有效期。不指定有效期时，cookie默认为当前session有效，当前session有效指关闭浏览器时cookie失效。

Cookie的有效期过长可能导致通行证泄露。比如用户使用了公用电脑，关闭浏览器时，没有点击注销。这样Cookie就留存在这台电脑上，其它人只需打开浏览器即可获取其在网站上的cookie。

一般保存通行证的cookie应该设置为session有效的cookie，并在用户选择记住登录状态时，提醒用户，不要再公用电脑上做这样的勾选。

### 明文传输的HTTP流量

HTTP明文传输数据的特性，使得攻击者可从网路上抓包获取Cookie。

解决方案：
- 服务端使用HTTPS
- 指定cookie的secure属性，该属性使cookie只能在HTTPS请求中带出。

### 利用XSS漏洞读取保存在cookie的通行证

假如网站存在XSS漏洞，那么恶意JS可直接读取Cookie中的通行证。可以通过指定Cookie的HttpOnly属性。对于指定了HttpOnly的Cookie，浏览器会拒绝JS读写。

XSS仍有可能利用服务端漏洞获取cookie：
    - 服务端有可能在请求的正常响应中包含通行证。不要笑，这真的有，尤其现在很多公司app和web共用一套后台；
    - 服务端可能有漏洞导致cookie包含在请求响应中，从而被窃取。比如：Apache CEV-2012-005

所以HttpOnly不是银弹，XSS也有很多其它的危害。但至少，HttpOnly可以避免通行证在浏览器被JS直接读取。

### 利用XSS漏洞写cookie

- 使用户退登

保存了通行证的cookie因为是HttpOnly，所以XSS无法读取。但XSS还是可以写Cookie，由于domain、path、name三者才唯一标识一个cookie，XSS可以写一个跟通行证cookie name相同，但domain或path不同的cookie。

这样当浏览器发送请求给服务端时，服务端就会收到两个相同name的cookie。因为cookie读时无属性，所以服务端无从判断哪一个才是正确的。这样会使用哪一个cookie，就要看具体web server的实现，可能是第一个也可能是第二个。

这样通过写cookie，XSS可以轻易地使已登录用户退登。

- 更危险的情况，Cookie替换攻击

恶意JS可针对特定的域名和path写入一个攻击者的cookie通行证。比如针对某电商网站的余额充值domain和path，写入攻击者的通行证。该攻击者cookie在用户正常购物的请求中不会带出，当用户充值时会带出，从而使用户充值到了攻击者的账号。

### 不安全的公共wifi

对于HTTP，公共wifi可劫持用户所有的流量，窃取cookie通行证自然不在话下。

对于HTTPS，因为可以在DNS上做手脚，可以在路由器上截取和篡改流量，在公共wifi上很容易诱导用户发出到网站的HTTP请求，几遍该网站是全站HTTPS也是如此。

因此，对于XSS写cookie的危害，公共wifi都能做到。而且能力更强，危害更大。

对于设置了secure属性的cookie，虽然不会在http请求中带出，但却可被http请求的Cookie设置覆盖。因此，设置了secure也不安全。

> Although seemingly useful for protecting cookies from active network
> attackers, the Secure attribute protects only the cookie's
> confidentiality.  An active network attacker can overwrite Secure
> cookies from an insecure channel, disrupting their integrity 
>                                     ---- from RFC 6265


### HSTS

公共wifi的危害，有一个关键点，就是诱导用户对指定网站发出HTTP请求。这一点通过[HSTS - HTTP Strict Transport Security](https://tools.ietf.org/html/rfc6797)可以避免。

HSTS通过设置strict‐transport‐security头，来告知浏览器，对当前域名以及所有子域名，都只使用HTTPS访问。

``` javascript

让浏览器对当前域名及子域名，强制进行HTTPS访问：

strict‐transport‐security: max‐age=15552000;includeSubDomains;

```

但HSTS在现实中难以被利用。因为：

- 囿于cookie的通配特性，必须对整个域名部署HSTS才能生效；
- 浏览器并非全支持这个特性。IE11+才开始支持；

所以，真正全站部署HSTS的网站，少之又少。

### 利用cookie大小和数量的限制来破坏安全

> o  At least 4096 bytes per cookie (as measured by the sum of the
      length of the cookie's name, value, and attributes).
> o  At least 50 cookies per domain.
> o  At least 3000 cookies total.
>                                     ---- From RFC 6265

- Cookie的大小和数量是有限制的；
- 每个浏览器，每种Web Server，限制都可能不一样；
- 超出限制的行为，是未定义的。

设置超大cookie，超多cookie，都是典型的攻击手段。上文提到的Apache CEV-2012-005漏洞，即是利用了Apache对Cookie大小的限制。

### 利用csrf漏洞偷用通行证

在访问恶意网站时，网站的页面代码中，可能包含到你银行账号或是其它什么账号的请求。由于这时浏览器会自动带出到相应域名的cookie，所以用户的登录通行证，就很容易被恶意网站偷用。

![csrf](http://zhaox.github.io/assets/images/csrf.png)

这个问题，可以通过设置Cookie的SameSite属性来解决。设置了SameSite属性的cookie，只会在当前正在访问（浏览器地址栏）的域名与cookie的domain可匹配时，才会带出。

- 在设置SameSite时，可以指定严格还是不严格。在不严格的情况下，如果请求会导致浏览器地址栏发生变化，那么cookie也是允许带出的；
- SameSite没有包含在RFC 6265中，因此还不是标准。但Chrome 51（发布于2016年8月）已经支持了这个属性。

如下截图，是Chrome支持的cookie属性：

![Chrome Cookie](http://zhaox.github.io/assets/images/ChromeCookie.png)

### 读取浏览器保存在硬盘上的cookie

有人会说，cookie不就是存放在硬盘，我直接从硬盘上窃取不可以吗？

答案是可以。比如Chrome的cookie就存放在目录：C:\Users\<username>\AppData\Local\Google\Chrome\User Data\Default\下，是一个sqllite的数据库文件，虽然cookie内容都进行了加密，但也是可以破解的。

只是，当你攻破一个用户的电脑，破解了chrome的加密算法，最终只是窃取到了一个用户的登录凭证而已。这种攻击，性价比很低。

### 举个例子

Github推出的github pages功能，一开始的时候使用的是github.com主站域名，后来因为上面提到的cookie安全问题，不得不为github pages单独启动了github.io域名。

具体可看Github的官方blog，里面提到了上面部分的cookie安全问题。blog链接：[Yummy Cookies across Domains](https://github.com/blog/1466-yummy-cookies-across-domains)

### 总结

- Cookie从机制上就有安全问题，使用时要小心再小心；
- 对于一些加强安全的Cookie属性，要尽量在系统早期实施。越晚实施，代价越大，实施难度越高；
- 公共wifi不安全，即便网站使用了https，公共wifi也不安全。当离开一个公共wifi时，清除这段时间浏览器产生的cookie！

### Reference

- [RFC 6265 - HTTP State Management Mechanism](https://tools.ietf.org/html/rfc6265)
- [HSTS - HTTP Strict Transport Security](https://tools.ietf.org/html/rfc6797)
- [Yummy Cookies across Domains](https://github.com/blog/1466-yummy-cookies-across-domains)
- [Cookie之困](https://github.com/knownsec/KCon/blob/master/2015/Cookie%20%E4%B9%8B%E5%9B%B0.pdf)
