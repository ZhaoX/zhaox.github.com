---
layout: post
title: "写给后端程序员的HTTP缓存原理介绍"
description: ""
category: Http 
tags: [Web, Http]
---
{% include JB/setup %}

> There are only two hard things in Computer Science: cache invalidation and naming things.

> -- Phil Karlton

通过Internet获取资源既缓慢，成本又高。为此，Http协议里包含了控制缓存的部分，以使Http客户端可以缓存和重用以前获取的资源，从而优化性能，提升体验。虽然Http中关于缓存控制的部分，随着协议演进，有一些变化。但我觉着，作为后端程序员，在开发Web服务时，只需要关注请求头If-None-Match、响应头ETag、响应头Cache-Control就足够了。因为这三个Http头就可以满足你的需求，并且，当今绝大多数的浏览器，都支持这三个Http头。我们所要做的就是，确保每个服务器响应都提供正确的 HTTP 头指令，以指导浏览器何时可以缓存响应以及可以缓存多久。

####缓存在哪儿？

![HttpCache](http://zhaox.github.io/assets/images/HttpCache.png)

上图中有三个角色，浏览器、Web代理和服务器，如图所示Http缓存存在于浏览器和Web代理中。当然在服务器内部，也存在着各种缓存，但这已经不是本文要讨论的Http缓存了。所谓的Http缓存控制，就是一种约定，通过设置不同的响应头Cache-Control来控制浏览器和Web代理对缓存的使用策略，通过设置请求头If-None-Match和响应头ETag，来对缓存的有效性进行验证。

####响应头ETag

ETag全称Entity Tag，用来标识一个资源。在具体的实现中，ETag可以是资源的hash值，也可以是一个内部维护的版本号。但不管怎样，ETag应该能反映出资源内容的变化，这是Http缓存可以正常工作的基础。

![HttpCacheEtag](http://zhaox.github.io/assets/images/HttpCacheEtag.png)

如上例中所展示的，服务器在返回响应时，通常会在Http头中包含一些关于响应的元数据信息，其中，ETag就是其中一个，本例中返回了值为x1323ddx的ETag。当资源/file的内容发生变化时，服务器应当返回不同的ETag。

####请求头If-None-Match

对于同一个资源，比如上一例中的/file，在进行了一次请求之后，浏览器就已经有了/file的一个版本的内容，和这个版本的ETag，当下次用户再需要这个资源，浏览器再次向服务器请求的时候，可以利用请求头If-None-Match来告诉服务器自己已经有个ETag为x1323ddx的/file，这样，如果服务器上的/file没有变化，也就是说服务器上的/file的ETag也是x1323ddx的话，服务器就不会再返回/file的内容，而是返回一个304的响应，告诉浏览器该资源没有变化，缓存有效。

![HttpCache](http://zhaox.github.io/assets/images/HttpCacheIfNoneMatch.png)

如上例中所示，在使用了If-None-Match之后，服务器只需要很小的响应就可以达到相同的结果，从而优化了性能。

####响应头Cache-Control

每个资源都可以通过Http头Cache-Control来定义自己的缓存策略，Cache-Control控制谁在什么条件下可以缓存响应以及可以缓存多久。 最快的请求是不必与服务器进行通信的请求：通过响应的本地副本，我们可以避免所有的网络延迟以及数据传输的数据成本。为此，HTTP 规范允许服务器返回一系列不同的 Cache-Control 指令，控制浏览器或者其他中继缓存如何缓存某个响应以及缓存多长时间。

> Cache-Control 头在 HTTP/1.1 规范中定义，取代了之前用来定义响应缓存策略的头（例如 Expires）。当前的所有浏览器都支持 Cache-Control，因此，使用它就够了。

以下我来介绍可以在Cache-Control中设置的常用指令。

######max-age
该指令指定从当前请求开始，允许获取的响应被重用的最长时间（单位为秒。例如：Cache-Control:max-age=60表示响应可以再缓存和重用 60 秒。需要注意的是，在max-age指定的时间之内，浏览器不会向服务器发送任何请求，包括验证缓存是否有效的请求，也就是说，如果在这段时间之内，服务器上的资源发生了变化，那么浏览器将不能得到通知，而使用老版本的资源。所以在设置缓存时间的长度时，需要慎重。

######public和private
如果设置了public，表示该响应可以再浏览器或者任何中继的Web代理中缓存，public是默认值，即Cache-Control:max-age=60等同于Cache-Control:public, max-age=60。

在服务器设置了private比如Cache-Control:private, max-age=60的情况下，表示只有用户的浏览器可以缓存private响应，不允许任何中继Web代理对其进行缓存 - 例如，用户浏览器可以缓存包含用户私人信息的 HTML 网页，但是 CDN 不能缓存。

######no-cache
如果服务器在响应中设置了no-cache即Cache-Control:no-cache，那么浏览器在使用缓存的资源之前，必须先与服务器确认返回的响应是否被更改，如果资源未被更改，可以避免下载。这个验证之前的响应是否被修改，就是通过上面介绍的请求头If-None-match和响应头ETag来实现的。
> 需要注意的是，no-cache这个名字有一点误导。设置了no-cache之后，并不是说浏览器就不再缓存数据，只是浏览器在使用缓存数据时，需要先确认一下数据是否还跟服务器保持一致。如果设置了no-cache，而ETag的实现没有反应出资源的变化，那就会导致浏览器的缓存数据一直得不到更新的情况。

######no-store
如果服务器在响应中设置了no-store即Cache-Control:no-store，那么浏览器和任何中继的Web代理，都不会存储这次相应的数据。当下次请求该资源时，浏览器只能重新请求服务器，重新从服务器读取资源。

####怎样决定一个资源的Cache-Control策略呢？

下面这个流程图，可以帮到你。

![HttpCache](http://zhaox.github.io/assets/images/HttpCacheControl.png)


####Reference
[https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching?hl=zh-cn)

