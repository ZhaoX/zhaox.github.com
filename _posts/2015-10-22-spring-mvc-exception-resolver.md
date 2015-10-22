---
layout: post
title: "Spring MVC异常处理详解"
description: ""
category: Java
tags: [Java, Web]
---
{% include JB/setup %}

#Spring MVC中异常处理的类体系机构

下图中，我画出了Spring MVC中，跟异常处理相关的主要类和接口。

![SpringMVCExceptionResolver](http://zhaox.github.io/assets/images/SpringMVCExceptionResolver.png)

在Spring MVC中，所有用于处理在请求映射和请求处理过程中抛出的异常的类，都要实现HandlerExceptionResolver接口。AbstractHandlerExceptionResolver实现该接口和Orderd接口，是HandlerExceptionResolver类的实现的基类。ResponseStatusExceptionResolver等具体的异常处理类均在AbstractHandlerExceptionResolver之上，实现了具体的异常处理方式。一个基于Spring MVC的Web应用程序中，可以存在多个实现了HandlerExceptionResolver的异常处理类，他们的执行顺序，由其order属性决定, order值越小，越是优先执行。

类图

逐个Resolver介绍


#Spring MVC是如何创建和使用这些Resolver的？


#何时该使用何种Exception Resolver？
