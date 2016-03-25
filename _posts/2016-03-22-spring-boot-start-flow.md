---
layout: post
title: "Spring Boot启动流程详解"
description: ""
category: Java 
tags: [Java, Web, Spring]
---
{% include JB/setup %}

### 环境

本文基于Spring Boot版本1.3.3, 使用了spring-boot-starter-web。

配置完成后，编写了代码如下：

``` java

@SpringBootApplication
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

@RestController
public class RootController {

    public static final String PATH_ROOT = "/";

    @RequestMapping(PATH_ROOT)
    public String welcome() {
        return "Welcome!";
    }

}

```
虽然只有几行代码，但是这已经是一个完整的Web程序，当访问url的path部分为"/"时，返回字符串"Welcome!"。

首先是一个非常普通的java程序入口，一个符合约定的静态main方法。在这个main方法中，调用了SpringApplication的静态run方法，并将Application类对象和main方法的参数args作为参数传递了进去。

然后是一个使用了两个Spring注解的RootController类，我们在main方法中，没有直接使用这个类。

### SpringApplication的静态run方法

``` java

public static ConfigurableApplicationContext run(Object source, String... args) {
	return run(new Object[] { source }, args);
}

public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
	return new SpringApplication(sources).run(args);
}

```

在这个静态方法中，创建SpringApplication对象，并调用该对象的run方法。

### 构造SpringApplication对象

``` java

public SpringApplication(Object... sources) {
	initialize(sources);
}

private void initialize(Object[] sources) {
	// 为成员变量sources赋值
	if (sources != null && sources.length > 0) {
		this.sources.addAll(Arrays.asList(sources));
	}
	this.webEnvironment = deduceWebEnvironment();
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	this.mainApplicationClass = deduceMainApplicationClass();
}

```

构造函数中调用initialize方法，初始化SpringApplication对象的成员变量sources，webEnvironment，initializers，listeners，mainApplicationClass。sources的赋值比较简单，就是我们传给SpringApplication.run方法的参数。剩下的几个，我们依次来看看是怎么做的。

首先是webEnvironment:

``` java

private boolean webEnvironment; 

private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };

private void initialize(Object[] sources) {
	...
        // 为成员变量webEnvironment赋值
        this.webEnvironment = deduceWebEnvironment();
	...
}

private boolean deduceWebEnvironment() {
	for (String className : WEB_ENVIRONMENT_CLASSES) {
		if (!ClassUtils.isPresent(className, null)) {
			return false;
		}
	}
	return true;
}

```
可以看到webEnvironment是一个boolean，该成员变量用来表示当前应用程序是不是一个Web应用程序。那么怎么决定当前应用程序是否Web应用程序呢，是通过在classpath中查看是否存在WEB_ENVIRONMENT_CLASSES这个数组中所包含的类，如果存在那么当前程序即是一个Web应用程序，反之则不然。

然后是initializers:

``` java

private List<ApplicationContextInitializer<?>> initializers;

private void initialize(Object[] sources) {
	...
	// 为成员变量sources赋值
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	...
}

public void setInitializers(
		Collection<? extends ApplicationContextInitializer<?>> initializers) {
	this.initializers = new ArrayList<ApplicationContextInitializer<?>>();
	this.initializers.addAll(initializers);
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
	return getSpringFactoriesInstances(type, new Class<?>[] {});
}

private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, Object... args) {
	ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
	// Use names and ensure unique to protect against duplicates
	Set<String> names = new LinkedHashSet<String>(
			SpringFactoriesLoader.loadFactoryNames(type, classLoader));
	List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
			classLoader, args, names);
	AnnotationAwareOrderComparator.sort(instances);
	return instances;
}

```
















