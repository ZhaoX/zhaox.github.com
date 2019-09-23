---
layout: post
title: "从单机应用到微服务，用户认证走几步？"
description: "End-user Authentication"
category: Web
tags: [Web, Authentication]
---
{% include JB/setup %}

用户认证指在用户访问服务的时候确认用户的身份，受限于HTTP无状态的特性，应用开发者需要自行实现用户认证相关功能。

通常是用户登录时服务端生成通行证返回给客户端，客户端在接下来的请求中携带通行证，然后服务端通过校验该通行证实现用户认证。

不过具体的业务是什么，如果用户认证失败，那么所有的后续操作都无法执行，需要返回给客户端用户认证失败，对应到HTTP Status Code是401。

本文阐述随着流量和规模增长，服务从单机应用发展到微服务架构的过程中，用户认证功能的实现方式变迁。

### 单机应用

![UserAuthSingle](http://zhaox.github.io/assets/images/UserAuthSingle.png)

这个阶段只有一台服务器，应用将session维护在本机的内存/磁盘，然后生成一个session id告之客户端即可。很多Web框架有内置的实现，开箱即用，有时间简单到你都没注意到用户认证的存在。

但是一台应用服务器往往不够用，有多个理由：流量太大一台扛不住、机器一挂服务就挂可用性太低...

### 负载均衡

![UserAuthLB](http://zhaox.github.io/assets/images/UserAuthLB.png)

因为有多台服务器，所以没法再简单地将session维护在本机上，因为用户在一台机器上登录后，下次访问可能就到了另一台机器。

- **方案A：认证令牌**

在登录的时候将一些用户的信息编码到一个字符串里返回给客户端，客户端在随后的操作中携带此字符串操作，服务端验证这个字符串，字符串合法则认证通过并从字符串中读取用户身份信息。

这样就不需要服务端再存储session信息，轻松支持多台服务器。而这个用来认证的字符串，就跟军队的通关令牌一样，我们管它叫认证令牌。

认证令牌的格式需要仔细设计，要能防篡改、具备有效期、要能放到URL中使用、还要可扩展等等。已经有人设计了好了一种令牌格式，还形成了标准叫JWT，可以直接拿来用，当然要是嫌JWT不好用，重新设计一个也是可以的。

这里有一份JWT的[简明介绍](http://zhaox.github.io/web/2019/09/20/jwt)。

认证令牌这个方案的缺点是，难以实现登录态的云端管理。通常一个令牌生成后，只能等它过期或者encode在其中的某些用户身份信息发生变化的时候才会生效。像用户退登就失效、用户的在线设备管理、用户的登录态管控等功能，没有办法实现。

- **方案B：分布式Session**

还有一种方案，将session从应用服务器拿出来单独维护，形成一个分布式session，这样每一台应用服务器都能访问得到。

![UserAuthLBSession](http://zhaox.github.io/assets/images/UserAuthLBSession.png)

具体到实现，分布式Session可以是Redis、MySQL等。但不管用什么样的存储系统，用户认证服务的可用性都强依赖它，相比认证令牌不需要服务端存储的方案，可用性肯定相对要低一些。

- **方案A+B**

认证令牌有功能缺失，分布式Session可用性相对不足，把两者结合起来就成了顺利成章的方案。

将认证令牌当做session id，正常服务时，用户认证都做分布式session校验，而当分布式session依赖的存储系统偶尔出现故障时，则服务降级，改为只校验认证令牌。这样可用性有保障，功能也齐全。

如果需要保证登录服务时，分布式Session也是高可用的，还可以在令牌里面增加一个字段用来区分有session和无session的类型。

### 微服务

业务规模越来越大，团队人数越来越多。一起维护同一个服务越来越难了，需要服务拆分，但不管怎么拆，用户认证是每个面向终端用户的服务都需要的。

![UserAuthSplit](http://zhaox.github.io/assets/images/UserAuthSplit.png)

用户认证都需要，只要在拆分的时候把相关代码都拷贝走，然后各个服务都要依赖用户信息和分布式session。

这样维护起来困难太大，只好将用户认证相关的功能拆分出来，形成一个独立的用户认证服务。这个服务几乎每一个大点的互联网公司都有，名字不太一样，有的叫Passport，有的叫Account，有的叫User，但总归功能是差不多的。

![UserAuthService](http://zhaox.github.io/assets/images/UserAuthService.png)

这样登录、注册、用户认证相关的代码、数据、线上部署，完全独立出来，形成一个用户认证服务。然后随着业务增长，服务会继续拆分，越来越多。服务之间可能有依赖，然后大部分服务尤其是面向终端用户的服务都会依赖用户认证服务。

![UserAuthMicroService](http://zhaox.github.io/assets/images/UserAuthMicroService.png)

依赖用户认证服务的独立微服务越来越多，重复的对接工作需要进行很多次，另外在用户的一次请求中往往涉及多个微服务，在各个微服务单独对接用户认证的情况下，难免会进行多次用户认证，造成重复请求，资源浪费。

### API网关

![UserAuthApiGateway](http://zhaox.github.io/assets/images/UserAuthApiGateway.png)

通过外网API网关，将公司所有的对外接口的签名和用户身份认证收拢到一起。API网关收到用户的认证令牌之后，先去用户认证服务换取用户id，然后使用用户id访问其它微服务。这样各个微服务都根据用户uid提供服务，不再需要关心终端用户的身份认证，而用户认证服务也只需要对接API网关即可，各自的工作量都大大减小。

### Reference

[https://www.slideshare.net/opencredo/authentication-in-microservice-systems-david-borsos](https://www.slideshare.net/opencredo/authentication-in-microservice-systems-david-borsos)