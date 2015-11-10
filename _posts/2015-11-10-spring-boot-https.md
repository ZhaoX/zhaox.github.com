---
layout: post
title: "在Spring Boot中使用Https"
description: ""
category: Java
tags: [Web, Spring]
---
{% include JB/setup %}

本文介绍如何在Spring Boot中，使用Https提供服务，并将Http请求自动重定向到Https。

###Https证书

巧妇难为无米之炊，开始的开始，要先取得Https证书。你可以向证书机构申请证书，也可以自己制作根证书。如果你对于Https的原理和证书制作还不清楚，可以看一下[Https原理介绍](http://zhaox.github.io/security/2015/11/06/explain-https/)和[自制Https证书](http://zhaox.github.io/security/2015/11/09/diy-https/)。

###创建Web配置类

在代码中创建一个使用了@Configuration注解的类，就像下面这段代码一样：

```java
@Configuration
public class WebConfig {
        //Bean 定义...
}
```

###配置Https

在配置类中添加EmbeddedServletContainerCustomizer Bean，并在其中配置Https证书和端口号。

```java
@Bean
public EmbeddedServletContainerCustomizer containerCustomizer() {
    return new EmbeddedServletContainerCustomizer() {
        @Override
        public void customize(ConfigurableEmbeddedServletContainer container) {
            Ssl ssl = new Ssl();
            //Server.jks中包含服务器私钥和证书
            ssl.setKeyStore("server.jks");
            ssl.setKeyStorePassword("123456");
            container.setSsl(ssl);
            container.setPort(8443);
        }
    };
}
```

###配置Http使其自动重定向到Https

Embedded默认只有一个Connector，要在提供Https服务的同时支持Http，需要添加一个Connector。在配置类中添加如下配置：

```java
@Bean
public EmbeddedServletContainerFactory servletContainerFactory() {
    TomcatEmbeddedServletContainerFactory factory =
        new TomcatEmbeddedServletContainerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                //SecurityConstraint必须存在，可以通过其为不同的URL设置不同的重定向策略。
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
    factory.addAdditionalTomcatConnectors(createHttpConnector());
    return factory;
}

private Connector createHttpConnector() {
    Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
    connector.setScheme("http");
    connector.setSecure(false);
    connector.setPort(8080);
    connector.setRedirectPort(8443);
    return connector;
}
```
