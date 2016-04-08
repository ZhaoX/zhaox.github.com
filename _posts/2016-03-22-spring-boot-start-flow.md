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

### SpringApplication类的静态run方法

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

下图画出了加载的ApplicationContextInitializer，并说明了他们的作用。至于何时应用他们，且听后面慢慢分解。

![SpringBootApplicationContextInitializer](http://zhaox.github.io/assets/images/SpringBootApplicationContextInitializer.png)


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

下图画出了加载的ApplicationListener，并说明了他们的作用。至于他们何时会被触发，等事件出现时，我们再说明。

![SpringBootApplicationListener](http://zhaox.github.io/assets/images/SpringBootApplicationListener.png)

### 最后是mainApplicationClass

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private Class<?> mainApplicationClass;

private void initialize(Object[] sources) {
	...
	// 为成员变量mainApplicationClass赋值
	this.mainApplicationClass = deduceMainApplicationClass();
	...
}

private Class<?> deduceMainApplicationClass() {
	try {
		StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
		for (StackTraceElement stackTraceElement : stackTrace) {
			if ("main".equals(stackTraceElement.getMethodName())) {
				return Class.forName(stackTraceElement.getClassName());
			}
		}
	}
	catch (ClassNotFoundException ex) {
		// Swallow and continue
	}
	return null;
}

```

在deduceMainApplicationClass方法中，通过获取当前调用栈，找到入口方法main所在的类，并将其复制给SpringApplication对象的成员变量mainApplicationClass。在我们的例子中mainApplicationClass即是我们自己编写的Application类。

### SpringApplication对象的run方法

经过上面的初始化过程，我们已经有了一个SpringApplication对象，根据SpringApplication类的静态run方法一节中的分析，接下来会调用SpringApplication对象的run方法。我们接下来就分析这个对象的run方法。

``` java

以下代码摘自：org.springframework.boot.SpringApplication

public ConfigurableApplicationContext run(String... args) {
	StopWatch stopWatch = new StopWatch();
	stopWatch.start();
	ConfigurableApplicationContext context = null;
	configureHeadlessProperty();
	SpringApplicationRunListeners listeners = getRunListeners(args);
	listeners.started();
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		context = createAndRefreshContext(listeners, applicationArguments);
		afterRefresh(context, applicationArguments);
		listeners.finished(context, null);
		stopWatch.stop();
		if (this.logStartupInfo) {
			new StartupInfoLogger(this.mainApplicationClass)
					.logStarted(getApplicationLog(), stopWatch);
		}
		return context;
	}
	catch (Throwable ex) {
		handleRunFailure(context, listeners, ex);
		throw new IllegalStateException(ex);
	}
}

```

- 可变个数参数args即是我们整个应用程序的入口main方法的参数，在我们的例子中，参数个数为零。

- StopWatch是来自org.springframework.util的工具类，可以用来方便的记录程序的运行时间。

SpringApplication对象的run方法创建并刷新ApplicationContext，算是开始进入正题了。下面按照执行顺序，介绍该方法所做的工作。

### headless模式

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private static final String SYSTEM_PROPERTY_JAVA_AWT_HEADLESS = "java.awt.headless";
private boolean headless = true;

public ConfigurableApplicationContext run(String... args) {
	...
	//设置headless模式
        configureHeadlessProperty();
	...
}

private void configureHeadlessProperty() {
	System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, System.getProperty(
			SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}

```

实际上是就是设置系统属性java.awt.headless，在我们的例子中该属性会被设置为true，因为我们开发的是服务器程序，一般运行在没有显示器和键盘的环境。关于java中的headless模式，更多信息可以参考[这里](http://www.oracle.com/technetwork/articles/javase/headless-136834.html)。

### SpringApplicationRunListeners


``` java

以下代码摘自：org.springframework.boot.SpringApplication

public ConfigurableApplicationContext run(String... args) {
	...
	SpringApplicationRunListeners listeners = getRunListeners(args);
	listeners.started();
	/**
         * 创建并刷新ApplicationContext
         * context = createAndRefreshContext(listeners, applicationArguments); 
        **/
	listeners.finished(context, null);
	...
}

private SpringApplicationRunListeners getRunListeners(String[] args) {
	Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
	return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
			SpringApplicationRunListener.class, types, this, args));
}

```

run方法中，加载了一系列SpringApplicationRunListener对象，在创建和更新ApplicationContext方法前后分别调用了listeners对象的started方法和finished方法, 并在创建和刷新ApplicationContext时，将listeners作为参数传递到了createAndRefreshContext方法中，以便在创建和刷新ApplicationContext的不同阶段，调用listeners的相应方法以执行操作。所以，所谓的SpringApplicationRunListeners实际上就是在SpringApplication对象的run方法执行的不同阶段，去执行一些操作，并且这些操作是可配置的。

同时，可以看到，加载SpringApplicationRunListener时，使用的是跟加载ApplicationContextInitializer和ApplicationListener时一样的方法。那么加载了什么，就可以从spring.factories文件中看到了：

``` java

以下内容摘自spring-boot-1.3.3.RELEASE.jar中的资源文件META-INF/spring.factories

# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener

```

可以看到，在我们的例子中加载的是org.springframework.boot.context.event.EventPublishingRunListener。我们看一看这个SpringApplicationRunListener究竟做了点什么工作了？

``` java

以下代码摘自：org.springframework.boot.context.event.EventPublishingRunListener

public EventPublishingRunListener(SpringApplication application, String[] args) {
	this.application = application;
	this.args = args;
	this.multicaster = new SimpleApplicationEventMulticaster();
	for (ApplicationListener<?> listener : application.getListeners()) {
		this.multicaster.addApplicationListener(listener);
	}
}

@Override
public void started() {
	publishEvent(new ApplicationStartedEvent(this.application, this.args));
}

@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
	publishEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args,
			environment));
}

@Override
public void contextPrepared(ConfigurableApplicationContext context) {
	registerApplicationEventMulticaster(context);
}

@Override
public void contextLoaded(ConfigurableApplicationContext context) {
	for (ApplicationListener<?> listener : this.application.getListeners()) {
		if (listener instanceof ApplicationContextAware) {
			((ApplicationContextAware) listener).setApplicationContext(context);
		}
		context.addApplicationListener(listener);
	}
	publishEvent(new ApplicationPreparedEvent(this.application, this.args, context));
}

@Override
public void finished(ConfigurableApplicationContext context, Throwable exception) {
	publishEvent(getFinishedEvent(context, exception));
}

```

EventPublishingRunListener在对象初始化时，将SpringApplication对象的成员变量listeners全都保存下来，然后在自己的public方法被调用时，发布相应的事件，或执行相应的操作。可以说这个RunListener是在SpringApplication对象的run方法执行到不同的阶段时，发布相应的event给SpringApplication对象的成员变量listeners中记录的事件监听器。

下图画出了SpringApplicationRunListeners相关的类结构，虽然我们的例子中只有一个SpringApplicationRunListener，但在这样的设计下，想要扩展是非常容易的！

![SpringBootApplicationContextInitializer](http://zhaox.github.io/assets/images/SpringBootSpringApplicationRunListener.png)

接下来，我们看一下在调用listeners的started方法。在我们的例子中，也就是发布了ApplicationStartedEvent时，我们已经加载的事件监听器都做了什么操作。至于其它事件的发布，我们按照代码执行的顺序在后面的章节在介绍。

- ParentContextCloserApplicationListener不监听ApplicationStartedEvent，没有操作；
- FileEncodingApplicationListener不监听ApplicationStartedEvent，没有操作；
- AnsiOutputApplicationListener不监听ApplicationStartedEvent，没有操作；
- ConfigFileApplicationListener不监听ApplicationStartedEvent，没有操作；
- DelegatingApplicationListener不监听ApplicationStartedEvent，没有操作；
- LiquibaseServiceLocatorApplicationListener监听ApplicationStartedEvent，会检查classpath中是否有liquibase.servicelocator.ServiceLocator并做相应操作；

``` java

以下代码摘自：org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener

@Override
public void onApplicationEvent(ApplicationStartedEvent event) {
	if (ClassUtils.isPresent("liquibase.servicelocator.ServiceLocator", null)) {
		new LiquibasePresent().replaceServiceLocator();
	}
}

```

我们的例子中，classpath中不存在liquibase，所以不执行任何操作。

- ClasspathLoggingApplicationListener监听ApplicationStartedEvent，会打印classpath到debug日志； 

``` java

@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationStartedEvent) {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Application started with classpath: " + getClasspath());
	}
	...
}

private String getClasspath() {
	ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
	if (classLoader instanceof URLClassLoader) {
		return Arrays.toString(((URLClassLoader) classLoader).getURLs());
	}
	return "unknown";
}

```

因为是debug级别的日志，而SpringBoot的默认日志级别是info级，所以我们在控制台不会看到classpath的输出。

- LoggingApplicationListener监听ApplicationStartedEvent，会根据classpath中的类情况创建相应的日志系统对象，并执行一些初始化之前的操作； 

``` java

@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationStartedEvent) {
		onApplicationStartedEvent((ApplicationStartedEvent) event);
	}
	...
}

private void onApplicationStartedEvent(ApplicationStartedEvent event) {
	this.loggingSystem = LoggingSystem
			.get(event.getSpringApplication().getClassLoader());
	this.loggingSystem.beforeInitialize();
}

```

我们的例子中，创建的是org.springframework.boot.logging.logback.LogbackLoggingSystem类的对象，Logback是SpringBoot默认采用的日志系统。下图画出了SpringBoot中的日志系统体系：

![SpringBootLoggingSystem](http://zhaox.github.io/assets/images/SpringBootLoggingSystem.png)

好了，ApplicationStartedEvent事件的处理这样就结束了。以后在介绍事件处理的时候，我们只介绍监听该事件的监听器的操作，而不监听的，就不再说明了。


### 创建并刷新ApplicationContext

``` java

以下代码摘自：org.springframework.boot.SpringApplication

public ConfigurableApplicationContext run(String... args) {
	...
	try {
		ApplicationArguments applicationArguments = new DefaultApplicationArguments(
				args);
		context = createAndRefreshContext(listeners, applicationArguments);
		afterRefresh(context, applicationArguments);
		...
	}
	catch (Throwable ex) {
		handleRunFailure(context, listeners, ex);
		throw new IllegalStateException(ex);
	}
}

```

首先是创建一个DefaultApplicationArguments对象，之后调用createAndRefreshContext方法创建并刷新一个ApplicationContext，最后调用afterRefresh方法在刷新之后做一些操作。

#### 先来看看DefaultApplicationArguments吧：

``` java 

以下代码摘自：org.springframework.boot.DefaultApplicationArguments

DefaultApplicationArguments(String[] args) {
	Assert.notNull(args, "Args must not be null");
	this.source = new Source(args);
	this.args = args;
}

private static class Source extends SimpleCommandLinePropertySource {

	Source(String[] args) {
		super(args);
	}
	...
}

以下代码摘自：org.springframework.core.env.SimpleCommandLinePropertySource

public SimpleCommandLinePropertySource(String... args) {
	super(new SimpleCommandLineArgsParser().parse(args));
}

```
可以看到是把main函数的args参数当做一个PropertySource来解析。我们的例子中，args的长度为0，所以这里创建的DefaultApplicationArguments也没有实际的内容。

#### 创建并配置ApplicationConext的Environment

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private ConfigurableEnvironment environment;

private boolean webEnvironment;

private ConfigurableApplicationContext createAndRefreshContext(
		SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments) {
	ConfigurableApplicationContext context;

	// 创建并配置Environment
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	configureEnvironment(environment, applicationArguments.getSourceArgs());
	listeners.environmentPrepared(environment);
	if (isWebEnvironment(environment) && !this.webEnvironment) {
		environment = convertToStandardEnvironment(environment);
	}

	...

	return context;
}

private ConfigurableEnvironment getOrCreateEnvironment() {
	if (this.environment != null) {
		return this.environment;
	}
	if (this.webEnvironment) {
		return new StandardServletEnvironment();
	}
	return new StandardEnvironment();
}

```

Spring Application的Environment代表着程序运行的环境，主要包含了两种信息，一种是profiles，用来描述哪些bean definitions是可用的；一种是properties，用来描述系统的配置，其来源可能是配置文件、JVM属性文件、操作系统环境变量等等。

首先要调用getOrCreateEnvironment方法获取一个Environment对象。在我们的例子中，执行到此处时，environment成员变量为null，而webEnvironment成员变量的值为true，所以会创建一个StandardServletEnvironment对象并返回。

之后是调用configureEnvironment方法来配置上一步获取的Environment对象，代码如下：

``` java

以下代码摘自：org.springframework.boot.SpringApplication

private Map<String, Object> defaultProperties;

private boolean addCommandLineProperties = true;

private Set<String> additionalProfiles = new HashSet<String>();

protected void configureEnvironment(ConfigurableEnvironment environment,
		String[] args) {
	configurePropertySources(environment, args);
	configureProfiles(environment, args);
}

protected void configurePropertySources(ConfigurableEnvironment environment,
		String[] args) {
	MutablePropertySources sources = environment.getPropertySources();
	if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
		sources.addLast(
				new MapPropertySource("defaultProperties", this.defaultProperties));
	}
	if (this.addCommandLineProperties && args.length > 0) {
		String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
		if (sources.contains(name)) {
			PropertySource<?> source = sources.get(name);
			CompositePropertySource composite = new CompositePropertySource(name);
			composite.addPropertySource(new SimpleCommandLinePropertySource(
					name + "-" + args.hashCode(), args));
			composite.addPropertySource(source);
			sources.replace(name, composite);
		}
		else {
			sources.addFirst(new SimpleCommandLinePropertySource(args));
		}
	}
}

protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
	environment.getActiveProfiles(); // ensure they are initialized
	// But these ones should go first (last wins in a property key clash)
	Set<String> profiles = new LinkedHashSet<String>(this.additionalProfiles);
	profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
	environment.setActiveProfiles(profiles.toArray(new String[profiles.size()]));
}

```

configureEnvironment方法先是调用configurePropertySources来配置properties，然后调用configureProfiles来配置profiles。

configurePropertySources首先查看SpringApplication对象的成员变量defaultProperties，如果该变量非null且内容非空，则将其加入到Environment的PropertySource列表的最后。然后查看SpringApplication对象的成员变量addCommandLineProperties和main函数的参数args，如果设置了addCommandLineProperties=true，且args个数大于0，那么就构造一个由main函数的参数组成的PropertySource放到Environment的PropertySource列表的最前面(这就能保证，我们通过main函数的参数来做的配置是最优先的，可以覆盖其他配置）。在我们的例子中，由于没有配置defaultProperties且main函数的参数args个数为0，所以这个函数什么也不做。

configureProfiles首先会读取Properties中key为spring.profiles.active的配置项，配置到Environment，然后再将SpringApplication对象的成员变量additionalProfiles加入到Environment的active profiles配置中。在我们的例子中，配置文件里没有spring.profiles.active的配置项，而SpringApplication对象的成员变量additionalProfiles也是一个空的集合，所以这个函数没有配置任何active profile。

到现在，Environment就算是配置完成了。接下来调用SpringApplicationRunListeners类的对象listeners发布ApplicationEnvironmentPreparedEvent事件：

``` java

以下代码摘自：org.springframework.boot.context.event.EventPublishingRunListener

@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
	publishEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args,
			environment));
}

```

好，现在来看一看我们加载的ApplicationListener对象都有哪些响应了这个事件，做了什么操作：

- FileEncodingApplicationListener响应该事件，检查file.encoding配置是否与spring.mandatory_file_encoding一致：

``` java

以下代码摘自：org.springframework.boot.context.FileEncodingApplicationListener

@Override
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
	RelaxedPropertyResolver resolver = new RelaxedPropertyResolver(
			event.getEnvironment(), "spring.");
	if (resolver.containsProperty("mandatoryFileEncoding")) {
		String encoding = System.getProperty("file.encoding");
		String desired = resolver.getProperty("mandatoryFileEncoding");
		if (encoding != null && !desired.equalsIgnoreCase(encoding)) {
			logger.error("System property 'file.encoding' is currently '" + encoding
					+ "'. It should be '" + desired
					+ "' (as defined in 'spring.mandatoryFileEncoding').");
			logger.error("Environment variable LANG is '" + System.getenv("LANG")
					+ "'. You could use a locale setting that matches encoding='"
					+ desired + "'.");
			logger.error("Environment variable LC_ALL is '" + System.getenv("LC_ALL")
					+ "'. You could use a locale setting that matches encoding='"
					+ desired + "'.");
			throw new IllegalStateException(
					"The Java Virtual Machine has not been configured to use the "
							+ "desired default character encoding (" + desired
							+ ").");
		}
	}
}

```

在我们的例子中，因为没有spring.mandatory_file_encoding的配置，所以这个响应方法什么都不做。

- AnsiOutputApplicationListener响应该事件，根据spring.output.ansi.enabled和spring.output.ansi.console-available对AnsiOutput类做相应配置:

``` java

以下代码摘自：org.springframework.boot.context.config.AnsiOutputApplicationListener

@Override
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
	RelaxedPropertyResolver resolver = new RelaxedPropertyResolver(
			event.getEnvironment(), "spring.output.ansi.");
	if (resolver.containsProperty("enabled")) {
		String enabled = resolver.getProperty("enabled");
		AnsiOutput.setEnabled(Enum.valueOf(Enabled.class, enabled.toUpperCase()));
	}

	if (resolver.containsProperty("console-available")) {
		AnsiOutput.setConsoleAvailable(
				resolver.getProperty("console-available", Boolean.class));
	}
}

```

我们的例子中，这两项配置都是空的，所以这个响应方法什么都不做。

- ConfigFileApplicationListener加载该事件，从一些约定的位置加载一些配置文件，而且这些位置是可配置的。

``` java

以下代码摘自：org.springframework.boot.context.config.ConfigFileApplicationListener

@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationEnvironmentPreparedEvent) {
		onApplicationEnvironmentPreparedEvent(
				(ApplicationEnvironmentPreparedEvent) event);
	}
	if (event instanceof ApplicationPreparedEvent) {
		onApplicationPreparedEvent(event);
	}
}

private void onApplicationEnvironmentPreparedEvent(
		ApplicationEnvironmentPreparedEvent event) {
	List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
	postProcessors.add(this);
	AnnotationAwareOrderComparator.sort(postProcessors);
	for (EnvironmentPostProcessor postProcessor : postProcessors) {
		postProcessor.postProcessEnvironment(event.getEnvironment(),
				event.getSpringApplication());
	}
}

List<EnvironmentPostProcessor> loadPostProcessors() {
	return SpringFactoriesLoader.loadFactories(EnvironmentPostProcessor.class,
			getClass().getClassLoader());
}


以下内容摘自spring-boot-1.3.3.RELEASE.jar中的资源文件META-INF/spring.factories

# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor

```

可以看到，ConfigFileApplicationListener从META-INF/spring.factories文件中读取EnvironmentPostProcessor配置，加载相应的EnvironmentPostProcessor类的对象，并调用其postProcessEnvironment方法。在我们的例子中，会加载CloudFoundryVcapEnvironmentPostProcessor和SpringApplicationJsonEnvironmentPostProcessor并执行，由于我们的例子中没有CloudFoundry和Json的配置，所以这个响应，不会加载任何的配置文件到Environment中来。

- DelegatingApplicationListener响应该事件，将配置文件中key为context.listener.classes的配置项，加载在成员变量multicaster中：

``` java

以下内容摘自：org.springframework.boot.context.config.DelegatingApplicationListener

private static final String PROPERTY_NAME = "context.listener.classes";

private SimpleApplicationEventMulticaster multicaster;

@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationEnvironmentPreparedEvent) {
		List<ApplicationListener<ApplicationEvent>> delegates = getListeners(
				((ApplicationEnvironmentPreparedEvent) event).getEnvironment());
		if (delegates.isEmpty()) {
			return;
		}
		this.multicaster = new SimpleApplicationEventMulticaster();
		for (ApplicationListener<ApplicationEvent> listener : delegates) {
			this.multicaster.addApplicationListener(listener);
		}
	}
	if (this.multicaster != null) {
		this.multicaster.multicastEvent(event);
	}
}

@SuppressWarnings("unchecked")
private List<ApplicationListener<ApplicationEvent>> getListeners(
		ConfigurableEnvironment env) {
	String classNames = env.getProperty(PROPERTY_NAME);
	List<ApplicationListener<ApplicationEvent>> listeners = new ArrayList<ApplicationListener<ApplicationEvent>>();
	if (StringUtils.hasLength(classNames)) {
		for (String className : StringUtils.commaDelimitedListToSet(classNames)) {
			try {
				Class<?> clazz = ClassUtils.forName(className,
						ClassUtils.getDefaultClassLoader());
				Assert.isAssignable(ApplicationListener.class, clazz, "class ["
						+ className + "] must implement ApplicationListener");
				listeners.add((ApplicationListener<ApplicationEvent>) BeanUtils
						.instantiateClass(clazz));
			}
			catch (Exception ex) {
				throw new ApplicationContextException(
						"Failed to load context listener class [" + className + "]",
						ex);
			}
		}
	}
	AnnotationAwareOrderComparator.sort(listeners);
	return listeners;
}

```

我们的例子中，因为没有key为context.listener.classes的Property，所以不会加载任何listener到该监听器中。

- LoggingApplicationListener响应该事件，并对在ApplicationStarted时加载的LoggingSystem做一些初始化工作：

``` java

以下代码摘自：org.springframework.boot.logging.LoggingApplicationListener

@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationStartedEvent) {
		onApplicationStartedEvent((ApplicationStartedEvent) event);
	}
	else if (event instanceof ApplicationEnvironmentPreparedEvent) {
		onApplicationEnvironmentPreparedEvent(
				(ApplicationEnvironmentPreparedEvent) event);
	}
	else if (event instanceof ApplicationPreparedEvent) {
		onApplicationPreparedEvent((ApplicationPreparedEvent) event);
	}
	else if (event instanceof ContextClosedEvent && ((ContextClosedEvent) event)
			.getApplicationContext().getParent() == null) {
		onContextClosedEvent();
	}
}

private void onApplicationEnvironmentPreparedEvent(
		ApplicationEnvironmentPreparedEvent event) {
	if (this.loggingSystem == null) {
		this.loggingSystem = LoggingSystem
				.get(event.getSpringApplication().getClassLoader());
	}
	initialize(event.getEnvironment(), event.getSpringApplication().getClassLoader());
}

protected void initialize(ConfigurableEnvironment environment,
		ClassLoader classLoader) {
	LogFile logFile = LogFile.get(environment);
	setSystemProperties(environment, logFile);
	initializeEarlyLoggingLevel(environment);
	initializeSystem(environment, this.loggingSystem, logFile);
	initializeFinalLoggingLevels(environment, this.loggingSystem);
	registerShutdownHookIfNecessary(environment, this.loggingSystem);
}

```

在我们的例子中，是对加载的LogbackLoggingSystem做一些初始化工作。关于日志系统更详细的讨论，值得再写一篇文章，就不在这里展开讨论了。






### 总结

- 扩展性好
