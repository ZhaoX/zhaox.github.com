---
layout: post
title: "REST API出错响应的设计"
description: ""
category: Design
tags: [REST, API]
---
{% include JB/setup %}

REST API应用很多，一方面提供公共API的平台越来越多，比如微博、微信等；一方面移动应用盛行，为Web端、Android端、IOS端、PC端，搭建一个统一的后台，以REST API的形式提供服务，也成为常见的开发模式。只是一个服务做得久了，就发现API的接口设计，如果能在一开始就好好设计一下，实在是功德无量的事。讨论API接口设计的文章已有不少，本文重点谈一谈当请求处理出现异常的时候，出错响应的内容和格式的设计。

比较自然的想法是，当有错误发生时，在响应中设置恰当的[HTTP Status Code](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)来指明这次请求是因为什么出错的。因为HTTP协议的标准性，绝大多数客户端可以理解收到的HTTP状态码，从而带来一致的体验。

但是，HTTP协议不可能定义出所有的出错可能，满足各种各样的服务的需求。事实上，用来表示请求出错的HTTP状态码只有24个，其中有18个4xx状态码用来表示客户端错误，6个5xx妆台码用来表示服务端错误。作为API的提供者，我们当然希望能够尽可能的规范我们的出错信息，提供更多的信息给调用者。因为API使用起来越是方便简单，客户就越有可能继续使用我们提供的服务，同时，当出现问题需要排查的时候，我们的工作也更加容易。

下面这个出错返回，列出了我认为错误信息里应当包含的内容。

``` javascript
{
    "status": 404,
    "code": 40483,
    "message": "Oops! It looks like that file does not exist.",
    "developerMessage": "File resource for path /uploads/foobar.txt does not exist.  Please wait 10 minutes until the upload batch completes before checking again.",
    "moreInfo": "http://www.mycompany.com/errors/40483",
    "requestId": "x3kdsa32k23ds32e"
}
```

###status
Status的内容与HTTP状态码内容相同，这个字段的存在，使得错误信息自包含，客户端只需要解析HTTP响应的body部分，就可以获取所有跟这次出错相关的信息。

###code
自定义错误码。自定义错误码的长度和个数都可以自己定义，这样就突破了HTTP状态码的个数限制。例子中的错误码是40483，其中404代表了请求的资源不存在，而83则制定了这次出错，具体是哪一种资源不存在。

#message
用户可理解的错误信息，应当根据用户的locale信息返回对应语言的版本。这个错误信息意在返回给使用客户端的用户阅读，不应该包含任何技术信息。有了这个字段，客户端的开发者在出错时，能够展示恰当的信息给最终用户。

#developerMessage
该出错的详细技术信息，提供给客户端的开发者阅读。可以包含Exception的信息、StackTrace，或者其它有用的技术信息。

#moreInfo
给出一个URL，客户端开发站访问这个URL可以看到更详细的关于该种出错信息的描述。在该URL展示的网页中，可以包含该出错信息的定义，产生原因，解决办法等等。

#requestId
请求ID，服务为每一个请求唯一生成一个请求ID，当客户端开发者无法自助解决问题时，可以联络服务开发者，同时提供该请求ID。一个好的服务，服务开发者应当可以根据此ID，定位到该次请求的所有相关log，进而定位问题，解决问题。


###Reference

https://stormpath.com/blog/spring-mvc-rest-exception-handling-best-practices-part-1/
