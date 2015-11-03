---
layout: post
title: "Spring Boot异常处理详解"
description: ""
category: Java
tags: [Java, Web, Spring]
---
{% include JB/setup %}

在[Spring MVC异常处理详解](http://zhaox.github.io/java/2015/10/22/spring-mvc-exception-resolver/)中，介绍了Spring MVC的异常处理体系，本文将讲解在此基础上Spring Boot为我们做了哪些工作。下图列出了Spring Boot中跟MVC异常处理相关的类。

![Spring Boot Web Exception Resolver](http://zhaox.github.io/assets/images/SpringBootWebExceptionResolver.png)

Spring Boot在启动过程中会根据当前环境进行AutoConfiguration，其中跟MVC错误处理相关的配置内容，在ErrorMvcAutoConfiguration这个类中。以下会分块介绍这个类里面的配置。

###在Servlet容器中添加了一个默认的错误页面

因为ErrorMvcAutoConfiguration类实现了EmbeddedServletContainerCustomizer接口，所以可以通过override customize方法来定制Servlet容器。以下代码摘自ErrorMvcAutoConfiguration：

``` java
@Value("${error.path:/error}")
private String errorPath = "/error";

@Override
public void customize(ConfigurableEmbeddedServletContainer container) {
    container.addErrorPages(new ErrorPage(this.properties.getServletPrefix()
        + this.errorPath));
}
```

可以看到ErrorMvcAutoConfiguration在容器中，添加了一个错误页面/error。因为这项配置的存在，如果Spring MVC在处理过程抛出异常到Servlet容器，容器会定向到这个错误页面/error。

那么我们有什么可以配置的呢？

1. 我们可以配置错误页面的url，/error是默认值，我们可以再application.properties中通过设置error.path的值来配置该页面的url；
2. 我们可以提供一个自定义的EmbeddedServletContainerCustomizer，添加更多的错误页面，比如对不同的http status code，使用不同的错误处理页面。就像下面这段代码一样：

``` java
@Bean
public EmbeddedServletContainerCustomizer containerCustomizer() {
    return new EmbeddedServletContainerCustomizer() {
        @Override
        public void customize(ConfigurableEmbeddedServletContainer container) {
            container.addErrorPages(new ErrorPage(HttpStatus.NOT_FOUND, "/404"));
            container.addErrorPages(new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/500"));
        }
    };
}
```

###定义了ErrorAttributes接口，并默认配置了一个DefaultErrorAttributes Bean

以下代码摘自ErrorMvcAutoConfiguration：

``` java
@Bean
@ConditionalOnMissingBean(value = ErrorAttributes.class, search = SearchStrategy.CURRENT)
public DefaultErrorAttributes errorAttributes() {
    return new DefaultErrorAttributes();
}
```

以下代码摘自DefaultErrorAttributes, ErrorAttributes, HandlerExceptionResolver：

``` java
@Order(Ordered.HIGHEST_PRECEDENCE)
public class DefaultErrorAttributes implements ErrorAttributes, HandlerExceptionResolver,
    Ordered {
    //篇幅原因，忽略类的实现代码。
}

public interface ErrorAttributes {
    public Map<String, Object> getErrorAttributes(RequestAttributes requestAttributes,
        boolean includeStackTrace);
    public Throwable getError(RequestAttributes requestAttributes);
}

public interface HandlerExceptionResolver {
    ModelAndView resolveException(
        HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```

这个DefaultErrorAttributes有什么用呢？主要有两个作用:

1. 实现了ErrorAttributes接口，具备提供Error Attributes的能力，当处理/error错误页面时，需要从该bean中读取错误信息以供返回；
2. 实现了HandlerExceptionResolver接口并具有最高优先级，即DispatcherServlet在doDispatch过程中有异常抛出时，先由DefaultErrorAttributes处理。从下面代码中可以发现，DefaultErrorAttributes在处理过程中，是讲ErrorAttributes保存到了request中。事实上，这是DefaultErrorAttributes能够在后面返回Error Attributes的原因，实现HandlerExceptionResolver接口，是DefaultErrorAttributes实现ErrorAttributes接口的手段。

``` java
@Override
public ModelAndView resolveException(HttpServletRequest request,
    HttpServletResponse response, Object handler, Exception ex) {
    storeErrorAttributes(request, ex);
    return null;
}
```

我们有什么可以配置的呢？
1、我们可以继承DefaultErrorAttributes，修改Error Attributes，比如下面这段代码，去掉了默认存在的error和exception这两个字段，以隐藏敏感信息。

``` java
@Bean
public DefaultErrorAttributes errorAttributes() {
    return new DefaultErrorAttributes() {
        @Override
        public Map<String, Object> getErrorAttributes (RequestAttributes requestAttributes,
        boolean includeStackTrace){
            Map<String, Object> errorAttributes = super.getErrorAttributes(requestAttributes, includeStackTrace);
            errorAttributes.remove("error");
            errorAttributes.remove("exception");
            return errorAttributes;
        }
    };
}
```

2. 我们可以自己实现ErrorAttributes接口，实现自己的Error Attributes方案, 只要配置一个类型为ErrorAttributes的bean，默认的DefaultErrorAttributes就不会被配置。


###提供并配置了ErrorController和ErrorView
ErrorController和ErrorView提供了对错误页面/error的支持。ErrorMvcAutoConfiguration默认配置了BasicErrorController和WhiteLabelErrorView，以下代码摘自ErrorMvcAutoConfiguration:

``` java
@Bean
@ConditionalOnMissingBean(value = ErrorController.class, search = SearchStrategy.CURRENT)
public BasicErrorController basicErrorController(ErrorAttributes errorAttributes) {
    return new BasicErrorController(errorAttributes);
}

@Configuration
@ConditionalOnProperty(prefix = "error.whitelabel", name = "enabled", matchIfMissing = true)
@Conditional(ErrorTemplateMissingCondition.class)
protected static class WhitelabelErrorViewConfiguration {
    private final SpelView defaultErrorView = new SpelView(
    		"<html><body><h1>Whitelabel Error Page</h1>"
    				+ "<p>This application has no explicit mapping for /error, so you are seeing this as a fallback.</p>"
    				+ "<div id='created'>${timestamp}</div>"
    				+ "<div>There was an unexpected error (type=${error}, status=${status}).</div>"
    				+ "<div>${message}</div></body></html>");

    @Bean(name = "error")
    @ConditionalOnMissingBean(name = "error")
    public View defaultErrorView() {
    	return this.defaultErrorView;
    }

    // If the user adds @EnableWebMvc then the bean name view resolver from
    // WebMvcAutoConfiguration disappears, so add it back in to avoid disappointment.
    @Bean
    @ConditionalOnMissingBean(BeanNameViewResolver.class)
    public BeanNameViewResolver beanNameViewResolver() {
    	BeanNameViewResolver resolver = new BeanNameViewResolver();
    	resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
    	return resolver;
    }
}
```

ErrorController根据Accept头的内容，输出不同格式的错误响应。比如针对浏览器的请求生成html页面，针对其它请求生成json格式的返回。代码如下：

``` java
@RequestMapping(value = "${error.path:/error}", produces = "text/html")
public ModelAndView errorHtml(HttpServletRequest request) {
    return new ModelAndView("error", getErrorAttributes(request, false));
}

@RequestMapping(value = "${error.path:/error}")
@ResponseBody
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {
    Map<String, Object> body = getErrorAttributes(request, getTraceParameter(request));
    HttpStatus status = getStatus(request);
    return new ResponseEntity<Map<String, Object>>(body, status);
}

```

WhitelabelErrorView则提供了一个默认的白板错误页面。

我们有什么可以配置的呢？
1. 我们可以提供自己的名字为error的view，以替换掉默认的白板页面，提供自己想要的样式。
2. 我们可以继承BasicErrorController或者干脆自己实现ErrorController接口，用来响应/error这个错误页面请求，可以提供更多类型的错误格式等。

###总结
Spring Boot提供了默认的统一错误页面，这是Spring MVC没有提供的。在理解了Spring Boot提供的错误处理相关内容之后，我们可以方便的定义自己的错误返回的格式和内容。不过，如果要实现统一的REST API接口的出错响应，就如[这篇文章](http://zhaox.github.io/design/2015/10/23/rest-api-error-response-design/)里的这样，还是要做不少工作的。
