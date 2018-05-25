---
layout: post
title: "跨域名登录态传递"
description: ""
category: Passport 
tags: [Web, Http, Cookie, SSO]
---
{% include JB/setup %}

很多互联网公司会有多个域名。这可能是因为公司并购，比如百度持有baidu.com、nuomi.com、qianqian.com等；也可能是为不同业务启用了不同的域名，比如阿里持有taobao.com、tmall.com等。

如果我们将用来实现登录接口的域名称之为主域名，其它域名称之为从域名，比如对百度来讲主域名是baidu.com，对阿里来讲主域名是taobao.com。那么用户在主域名登录后，自动将登录态同步到从域名，就成为一个很自然的需求。本文介绍如何安全高效地实现这种跨域名的登录态传递。

### 约束

1. 浏览器安全策略不允许跨域种cookie；
2. 部分浏览器（safari、firefox）默认禁止第三方cookie；
3. web端登录态标识（authcookie）在存储和传输时，不应该出现在cookie以外的任何地方，应设置cookie属性httponly为true；

### 基于push的方案

所谓push指的是主域名登录后，在主域名的页面下通过访问从域名的sso接口种cookie，将登录态“推送”到从域名；

主域名登录后，访问主域名sso接口获取token，然后将token拼接到query string中访问从域名下的sso接口，从域名的sso接口验证token并其对应的登录态标识种到从域名下的cookie中。

![CrossDomainSSOPush](http://zhaox.github.io/assets/images/CrossDomainSSOPush.png)

需要注意的是，由于约束2的存在，访问从域名的sso接口时，除非跳转页面或弹窗，否则在safari和firefox下因种三方cookie的行为被禁止，登录态无法传递成功；

### 基于pull的方案

pull指的是访问从域名的页面时，可访问主域名的sso接口，该接口依据主域名的登录态创建sso token，随后校验referer再跳转到对应从域名的sso接口。从域名的sso接口验证token并其对应的登录态标识种到从域名下的cookie中。这样就实现了将主域名的登录态“拉取”到了从域名下。

![CrossDomainSSOPull](http://zhaox.github.io/assets/images/CrossDomainSSOPull.png)

pull方案可以满足本文开头提出的三个约束，缺点是非登录相关页面也将包含登录态传递逻辑。

### 总结

跨域名sso主要涉及到一些cookie的知识点，不清楚的可以看[这篇文章](http://zhaox.github.io/security/2017/02/11/cookie-and-passport-security)。建议同时实现push和pull两种方案。