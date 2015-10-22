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

在Spring MVC中，所有用于处理在请求映射和请求处理过程中抛出的异常的类，都要实现HandlerExceptionResolver接口。AbstractHandlerExceptionResolver实现该接口和Orderd接口，是HandlerExceptionResolver类的实现的基类。ResponseStatusExceptionResolver等具体的异常处理类均在AbstractHandlerExceptionResolver之上，实现了具体的异常处理方式。一个基于Spring MVC的Web应用程序中，可以存在多个实现了HandlerExceptionResolver的异常处理类，他们的执行顺序，由其order属性决定, order值越小，越是优先执行, 在执行到第一个返回不是null的ModelAndView的Resolver时，不再执行后续的尚未执行的Resolver的异常处理方法。。

下面我逐个介绍一下SpringMVC提供的这些异常处理类的功能。

###DefaultHandlerExceptionResolver

HandlerExceptionResolver接口的默认实现，基本上是Spring MVC内部使用，用来处理Spring定义的各种标准异常，将其转化为相对应的HTTP Status Code。其处理的异常类型有：

``` java
handleNoSuchRequestHandlingMethod
handleHttpRequestMethodNotSupported
handleHttpMediaTypeNotSupported
handleMissingServletRequestParameter
handleServletRequestBindingException
handleTypeMismatch
handleHttpMessageNotReadable
handleHttpMessageNotWritable
handleMethodArgumentNotValidException
handleMissingServletRequestParameter
handleMissingServletRequestPartException
handleBindException
```

###ResponseStatusExceptionResolver

用来支持@ResponseStatus的使用，处理使用了ResponseStatus注解的异常，根据注解的内容，返回相应的HTTP Status Code和内容给客户端。如果Web应用程序中配置了ResponseStatusExceptionResolver，那么我们就可以使用ResponseStatus注解来注解我们自己编写的异常类，并在Controller中抛出该异常类，之后ResponseStatusExceptionResolver就会自动帮我们处理剩下的工作。

这是一个自己编写的异常，用来表示订单不存在：

``` java
 @ResponseStatus(value=HttpStatus.NOT_FOUND, reason="No such Order")  // 404
    public class OrderNotFoundException extends RuntimeException {
        // ...
    }
```

这是一个使用该异常的Controller方法：

``` java
@RequestMapping(value="/orders/{id}", method=GET)
    public String showOrder(@PathVariable("id") long id, Model model) {
        Order order = orderRepository.findOrderById(id);
        if (order == null) throw new OrderNotFoundException(id);
        model.addAttribute(order);
        return "orderDetail";
    }
```

这样，当OrderNotFoundException被抛出时，ResponseStatusExceptionResolver会返回给客户端一个HTTP Status Code为404的响应。


###AnnotationMethodHandlerExceptionResolver和ExceptionHandlerExceptionResolver
用来支持@ExceptionHandler注解，使用被ExceptionHandler注解所标记的方法来处理异常。其中AnnotationMethodHandlerExceptionResolver在3.0版本中开始提供，ExceptionHandlerExceptionResolver在3.1版本中开始提供，从3.2版本开始，Spring推荐使用ExceptionHandlerExceptionResolver。
如果配置了AnnotationMethodHandlerExceptionResolver和ExceptionHandlerExceptionResolver这两个异常处理bean之一，那么我们就可以使用ExceptionHandler注解来处理异常。

下面是几个ExceptionHandler注解的使用例子：

``` java
@Controller
public class ExceptionHandlingController {

  // @RequestHandler methods
  ...
  
  // 以下是异常处理方法
  
  // 将DataIntegrityViolationException转化为Http Status Code为409的响应
  @ResponseStatus(value=HttpStatus.CONFLICT, reason="Data integrity violation")  // 409
  @ExceptionHandler(DataIntegrityViolationException.class)
  public void conflict() {
    // Nothing to do
  }
  
  // 针对SQLException和DataAccessException返回视图databaseError
  @ExceptionHandler({SQLException.class,DataAccessException.class})
  public String databaseError() {
    // Nothing to do.  Returns the logical view name of an error page, passed to
    // the view-resolver(s) in usual way.
    // Note that the exception is _not_ available to this view (it is not added to
    // the model) but see "Extending ExceptionHandlerExceptionResolver" below.
    return "databaseError";
  }

  // 创建ModleAndView，将异常和请求的信息放入到Model中，指定视图名字，并返回该ModleAndView
  @ExceptionHandler(Exception.class)
  public ModelAndView handleError(HttpServletRequest req, Exception exception) {
    logger.error("Request: " + req.getRequestURL() + " raised " + exception);

    ModelAndView mav = new ModelAndView();
    mav.addObject("exception", exception);
    mav.addObject("url", req.getRequestURL());
    mav.setViewName("error");
    return mav;
  }
}
```

需要注意的是，上面例子中的ExceptionHandler方法的作用域，只是在本Controller类中。如果需要使用ExceptionHandler来处理全局的Exception，则需要使用@ControllerAdvice注解。

``` java
@ControllerAdvice
class GlobalDefaultExceptionHandler {
    public static final String DEFAULT_ERROR_VIEW = "error";

    @ExceptionHandler(value = Exception.class)
    public ModelAndView defaultErrorHandler(HttpServletRequest req, Exception e) throws Exception {
        // 如果异常使用了ResponseStatus注解，那么重新抛出该异常，Spring框架会处理该异常。 
        if (AnnotationUtils.findAnnotation(e.getClass(), ResponseStatus.class) != null)
            throw e;

        // 否则创建ModleAndView，处理该异常。
        ModelAndView mav = new ModelAndView();
        mav.addObject("exception", e);
        mav.addObject("url", req.getRequestURL());
        mav.setViewName(DEFAULT_ERROR_VIEW);
        return mav;
    }
}
```

###SimpleMappingExceptionResolver
提供了将异常映射为视图的能力，高度可定制化。其提供的能力有：
1. 根据异常的类型，将异常映射到视图；
2. 可以为不符合处理条件没有被处理的异常，指定一个默认的错误返回；
3. 处理异常时，记录log信息；
4. 指定需要添加到Modle中的Exception属性，从而在视图中展示该属性。

``` java
@Configuration
@EnableWebMvc 
public class MvcConfiguration extends WebMvcConfigurerAdapter {
    @Bean(name="simpleMappingExceptionResolver")
    public SimpleMappingExceptionResolver createSimpleMappingExceptionResolver() {
        SimpleMappingExceptionResolver r = new SimpleMappingExceptionResolver();

        Properties mappings = new Properties();
        mappings.setProperty("DatabaseException", "databaseError");
        mappings.setProperty("InvalidCreditCardException", "creditCardError");

        r.setExceptionMappings(mappings);  // 默认为空
        r.setDefaultErrorView("error");    // 默认没有
        r.setExceptionAttribute("ex"); 
        r.setWarnLogCategory("example.MvcLogger"); 
        return r;
    }
    ...
}
```


###自定义ExceptionResolver
Spring MVC的异常处理非常的灵活，如果提供的ExceptionResolver类不能满足使用，我们可以实现自己的异常处理类。可以通过继承SimpleMappingExceptionResolver来定制Mapping的方式和能力，也可以直接继承AbstractHandlerExceptionResolver来实现其它类型的异常处理类。


#Spring MVC是如何创建和使用这些Resolver的？


#何时该使用何种Exception Resolver？
