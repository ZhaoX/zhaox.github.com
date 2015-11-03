---
layout: post
title: "Spring MVC异常处理详解"
description: ""
category: Java
tags: [Java, Web, Spring]
---
{% include JB/setup %}

#Spring MVC中异常处理的类体系结构

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

首先看Spring MVC是怎么加载异常处理bean的。

1. Spring MVC有两种加载异常处理类的方式，一种是根据类型，这种情况下，会加载ApplicationContext下所有实现了ExceptionResolver接口的bean，并根据其order属性排序，依次调用；一种是根据名字，这种情况下会加载ApplicationContext下，名字为handlerExceptionResolver的bean。
2. 不管使用那种加载方式，如果在ApplicationContext中没有找到异常处理bean，那么Spring MVC会加载默认的异常处理bean。
3. 默认的异常处理bean定义在DispatcherServlet.properties中。

``` java
org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.annotation.AnnotationMethodHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver
```

以下代码摘自ispatcherServlet，描述了异常处理类的加载过程：

``` java
/**
 * Initialize the HandlerMappings used by this class.
 * <p>If no HandlerMapping beans are defined in the BeanFactory for this namespace,
 * we default to BeanNameUrlHandlerMapping.
 */
private void initHandlerMappings(ApplicationContext context) {
	this.handlerMappings = null;

	if (this.detectAllHandlerMappings) {
		// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
		Map<String, HandlerMapping> matchingBeans =
				BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
		if (!matchingBeans.isEmpty()) {
			this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
			// We keep HandlerMappings in sorted order.
			OrderComparator.sort(this.handlerMappings);
		}
	}
	else {
		try {
			HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
			this.handlerMappings = Collections.singletonList(hm);
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Ignore, we'll add a default HandlerMapping later.
		}
	}

	// Ensure we have at least one HandlerMapping, by registering
	// a default HandlerMapping if no other mappings are found.
	if (this.handlerMappings == null) {
		this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
		if (logger.isDebugEnabled()) {
			logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
		}
	}
}
```

然后看Spring MVC是怎么使用异常处理bean的。

1. Spring MVC把请求映射和处理过程放到try catch中，捕获到异常后，使用异常处理bean进行处理。
2. 所有异常处理bean按照order属性排序，在处理过程中，遇到第一个成功处理异常的异常处理bean之后，不再调用后续的异常处理bean。

以下代码摘自DispatcherServlet，描述了处理异常的过程。

``` java
/**
 * Process the actual dispatching to the handler.
 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
 * to find the first that supports the handler class.
 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
 * themselves to decide which methods are acceptable.
 * @param request current HTTP request
 * @param response current HTTP response
 * @throws Exception in case of any kind of processing failure
 */
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HttpServletRequest processedRequest = request;
	HandlerExecutionChain mappedHandler = null;
	boolean multipartRequestParsed = false;

	WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

	try {
		ModelAndView mv = null;
		Exception dispatchException = null;

		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);

			// Determine handler for the current request.
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}

			// Determine handler adapter for the current request.
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

			// Process last-modified header, if supported by the handler.
			String method = request.getMethod();
			boolean isGet = "GET".equals(method);
			if (isGet || "HEAD".equals(method)) {
				long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
				if (logger.isDebugEnabled()) {
					logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
				}
				if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
					return;
				}
			}

			if (!mappedHandler.applyPreHandle(processedRequest, response)) {
				return;
			}

			// Actually invoke the handler.
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}

			applyDefaultViewName(request, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	catch (Exception ex) {
		triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
	}
	catch (Error err) {
		triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
	}
	finally {
		if (asyncManager.isConcurrentHandlingStarted()) {
			// Instead of postHandle and afterCompletion
			if (mappedHandler != null) {
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
			}
		}
		else {
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
}


/**
 * Determine an error ModelAndView via the registered HandlerExceptionResolvers.
 * @param request current HTTP request
 * @param response current HTTP response
 * @param handler the executed handler, or {@code null} if none chosen at the time of the exception
 * (for example, if multipart resolution failed)
 * @param ex the exception that got thrown during handler execution
 * @return a corresponding ModelAndView to forward to
 * @throws Exception if no error ModelAndView found
 */
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
		Object handler, Exception ex) throws Exception {

	// Check registered HandlerExceptionResolvers...
	ModelAndView exMv = null;
	for (HandlerExceptionResolver handlerExceptionResolver : this.handlerExceptionResolvers) {
		exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
		if (exMv != null) {
			break;
		}
	}
	if (exMv != null) {
		if (exMv.isEmpty()) {
			request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
			return null;
		}
		// We might still need view name translation for a plain error model...
		if (!exMv.hasView()) {
			exMv.setViewName(getDefaultViewName(request));
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
		}
		WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
		return exMv;
	}

	throw ex;
}
```

#何时该使用何种ExceptionResolver？

Spring提供了很多选择和非常灵活的使用方式，下面是一些使用建议：

1. 如果自定义异常类，考虑加上@ResponseStatus注解；
2. 对于没有@ResponseStatus注解的异常，可以通过使用@ExceptionHandler+@ControllerAdvice，或者通过配置SimpleMappingExceptionResolver，来为整个Web应用提供统一的异常处理。
3. 如果应用中有些异常处理方式，只针对特定的Controller使用，那么在这个Controller中使用@ExceptionHandler注解。
4. 不要使用过多的异常处理方式，不然的话，维护起来会很苦恼，因为异常的处理分散在很多不同的地方。

