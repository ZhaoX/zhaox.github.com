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

以下代码摘自：org.springframework.boot.SpringApplication

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

以下代码摘自：org.springframework.boot.SpringApplication

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

### 首先是webEnvironment:

``` java

以下代码摘自：org.springframework.boot.SpringApplication

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
在本文的例子中webEnvironment的值为true。

### 然后是initializers:

initializers成员变量，是一个ApplicationContextInitializer类型对象的集合。 顾名思义，ApplicationContextInitializer是一个可以用来初始化ApplicationContext的接口。

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private List<ApplicationContextInitializer<?>> initializers;

private void initialize(Object[] sources) {
	...
	// 为成员变量initializers赋值
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	...
}

public void setInitializers(
		Collection<? extends ApplicationContextInitializer<?>> initializers) {
	this.initializers = new ArrayList<ApplicationContextInitializer<?>>();
	this.initializers.addAll(initializers);
}

```

可以看到，关键是调用getSpringFactoriesInstances(ApplicationContextInitializer.class)，来获取ApplicationContextInitializer类型对象的列表。

``` java

以下代码摘自：org.springframework.boot.SpringApplication

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

在该方法中，首先通过调用SpringFactoriesLoader.loadFactoryNames(type, classLoader)来获取所有Spring Factories的名字，然后调用createSpringFactoriesInstances方法根据读取到的名字创建对象。最后会将创建好的对象列表排序并返回。

``` java

以下代码摘自：org.springframework.core.io.support.SpringFactoriesLoader

public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
	String factoryClassName = factoryClass.getName();
	try {
		Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
				ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		List<String> result = new ArrayList<String>();
		while (urls.hasMoreElements()) {
			URL url = urls.nextElement();
			Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
			String factoryClassNames = properties.getProperty(factoryClassName);
			result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
		}
		return result;
	}
	catch (IOException ex) {
		throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
				"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
	}
}

```

可以看到，是从一个名字叫spring.factories的资源文件中，读取key为org.springframework.context.ApplicationContextInitializer的value。而spring.factories的部分内容如下：

``` java

以下内容摘自spring-boot-1.3.3.RELEASE.jar中的资源文件META-INF/spring.factories

# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.context.web.ServerPortInfoApplicationContextInitializer

```

可以看到，最近的得到的，是ConfigurationWarningsApplicationContextInitializer，ContextIdApplicationContextInitializer，DelegatingApplicationContextInitializer，ServerPortInfoApplicationContextInitializer这四个类的名字。

接下来会调用createSpringFactoriesInstances来创建ApplicationContextInitializer实例。

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private <T> List<T> createSpringFactoriesInstances(Class<T> type,
		Class<?>[] parameterTypes, ClassLoader classLoader, Object[] args,
		Set<String> names) {
	List<T> instances = new ArrayList<T>(names.size());
	for (String name : names) {
		try {
			Class<?> instanceClass = ClassUtils.forName(name, classLoader);
			Assert.isAssignable(type, instanceClass);
			Constructor<?> constructor = instanceClass.getConstructor(parameterTypes);
			T instance = (T) constructor.newInstance(args);
			instances.add(instance);
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException(
					"Cannot instantiate " + type + " : " + name, ex);
		}
	}
	return instances;
}

```

所以在我们的例子中，SpringApplication对象的成员变量initalizers就被初始化为，ConfigurationWarningsApplicationContextInitializer，ContextIdApplicationContextInitializer，DelegatingApplicationContextInitializer，ServerPortInfoApplicationContextInitializer这四个类的对象组成的list。


### 接下来是成员变量listeners

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private List<ApplicationListener<?>> listeners;

private void initialize(Object[] sources) {
	...
	// 为成员变量listeners赋值
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	...
}

public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
	this.listeners = new ArrayList<ApplicationListener<?>>();
	this.listeners.addAll(listeners);
}

```

listeners成员变量，是一个ApplicationListener<?>类型对象的集合。可以看到获取该成员变量内容使用的是跟成员变量initializers一样的方法，只不过传入的类型从ApplicationContextInitializer.class变成了ApplicationListener.class。

看一下spring.factories中的相关内容：

``` java

以下内容摘自spring-boot-1.3.3.RELEASE.jar中的资源文件META-INF/spring.factories

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.logging.LoggingApplicationListener

```

也就是说，在我们的例子中，listener最终会被初始化为ParentContextCloserApplicationListener，FileEncodingApplicationListener，AnsiOutputApplicationListener，ConfigFileApplicationListener，DelegatingApplicationListener，LiquibaseServiceLocatorApplicationListener，ClasspathLoggingApplicationListener，LoggingApplicationListener这几个类的对象组成的list。




















