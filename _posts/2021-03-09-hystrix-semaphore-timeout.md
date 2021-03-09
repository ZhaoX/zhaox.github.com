---
layout: post
title: "Hystrix信号量隔离模式下的超时时间配置"
description: ""
category: Hystrix
tags: [Hystrix, Timeout]
---
{% include JB/setup %}

### 隔离策略选择信号量时，还能配置超时时间吗？

可以配置

注解方式：

``` java
@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "2000")
代码方式：

HystrixCommandProperties.Setter()
   .withExecutionTimeoutInMilliseconds(int value)
```

时间单位都是毫秒

### 配置之后能生效吗?

生效，也不生效。下面分两种情况说明。

假设有如下Command：

``` java
  public static HystrixCommand.Setter getTestCommandConfig() {
    String commandName = "TestCommand";

    HystrixCommand.Setter commandConfig = HystrixCommand.Setter
            .withGroupKey(HystrixCommandGroupKey.Factory.asKey(commandName))
            .andCommandKey(HystrixCommandKey.Factory.asKey(commandName))
            .andCommandPropertiesDefaults(
                    HystrixCommandProperties.Setter()
                            .withExecutionIsolationStrategy(ExecutionIsolationStrategy.SEMAPHORE)
                            .withExecutionTimeoutInMilliseconds(100));

    return commandConfig;
  }

  static class TestCommand extends HystrixCommand<String> {

    protected TestCommand(Setter setter) {
      super(setter);
    }

    @Override
    protected String run() throws Exception {
      System.out.println("RUN start in " + Thread.currentThread().getName());
      Thread.sleep(1000);
      System.out.println("RUN end in " + Thread.currentThread().getName());
      return "HELLO from " + Thread.currentThread().getName();
    }

  }

}
```

##### 如果直接调用Command对象的execute方法：

new TestCommand(getTestCommandConfig()).execute()
那么command的run方法，会在调用execute()方法的线程中执行，并一直到run方法执行完成。

run方法执行完成之后，如果耗时没有超过配置的超时时间，那么执行结果正常返回。

如果耗时超过了超时时间，那么执行结果不会正常返回，而是抛出HystrixRuntimException，虽然run方法已经正常执行完了。

``` java
Exception in thread "main" com.netflix.hystrix.exception.HystrixRuntimeException: TestCommand timed-out and no fallback available.
```

##### 如果将Command对象转为Observable再执行：

``` java
new TestCommand(getTestCommandConfig()).toObservable().subscribe(new Observer<String>() {
      public void onCompleted() {

      }

      public void onError(Throwable e) {
        System.out.println(Thread.currentThread().getName() + " received " + e.getMessage());
        e.printStackTrace();
      }

      public void onNext(String s) {
        System.out.println(Thread.currentThread().getName() + " received " + s);
      }
    });
``` 

那么当前线程会被阻塞，直到run方法执行完成。

如果执行时间没有超时，那么onNext方法会被当前线程调用，参数是run方法的执行结果。

如果执行时间超时了，那么observable的onError方法，会在run方法执行到command的超时时间时被调用。onError方法是在名为HystrixTimer-1线程中执行的，这个线程是Hystrix自己创建和管理的。同时run方法执行完成后，onNext方法不会再被调用。

``` java
HystrixTimer-1 received TestCommand timed-out and no fallback available.
com.netflix.hystrix.exception.HystrixRuntimeException: TestCommand timed-out and no fallback available.
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:819)
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:804)
```

也就是说，虽然当前线程都会被阻塞。但是使用observable这种方式调用，通过编写合适的onError方法，可以在command执行超时时，立刻做一些事情。


### 配置了Fallback之后会怎样？

给Command加上fallback方法：

``` java
  static class TestCommand extends HystrixCommand<String> {

    protected TestCommand(Setter setter) {
      super(setter);
    }

    @Override
    protected String run() throws Exception {
      System.out.println("RUN start in " + Thread.currentThread().getName());
      Thread.sleep(5000);
      System.out.println("RUN end in " + Thread.currentThread().getName());
      return "HELLO from " + Thread.currentThread().getName();
    }

    @Override
    protected String getFallback() {
      System.out.println("Fallback in " + Thread.currentThread().getName());
      return "Fallback from " + Thread.currentThread().getName();
    }
  }
``` 


##### 如果直接调用Command对象的execute方法：

new TestCommand(getTestCommandConfig()).execute()
那么command的run方法，会在调用execute()方法的线程中执行，并一直到run方法执行完成。

run方法执行完成之后，如果耗时没有超过配置的超时时间，那么执行结果正常返回。

如果耗时超过了超时时间，那么在执行耗时达到了配置的超时时间的那一刻，getFallback方法会被HystrixTimer-1线程调用。然后等run方法执行完成后，execute方法不会返回run方法的执行结果，而是返回getFallback方法的执行结果。

也就是说run方法和getFallback方法会同时执行，run方法在调用execute方法的线程中执行，getFallback方法在超时时在HystrixTimer-1线程中执行。



##### 如果将Command对象转为Observable再执行：

``` java
new TestCommand(getTestCommandConfig()).toObservable().subscribe(new Observer<String>() {
      public void onCompleted() {

      }

      public void onError(Throwable e) {
        System.out.println(Thread.currentThread().getName() + " received " + e.getMessage());
        e.printStackTrace();
      }

      public void onNext(String s) {
        System.out.println(Thread.currentThread().getName() + " received " + s);
      }
    });
```

那么当前线程会被阻塞，直到run方法执行完成。

如果执行时间没有超时，那么onNext方法会被当前线程调用，参数是run方法的执行结果。

如果执行时间超时了，那么Command的getFallback方法和observable的onNext方法，会在run方法执行到command的超时时间时被调用，在HystrixTimer-1线程中执行。onNext方法拿到getFallback方法的执行结果，同时run方法执行完成后，onNext方法不会再被调用。



### 应该怎么配置？

虽然配置了超时时间，hystrix可以做一些事情，比如在执行耗时达到超时时间时，及时调用getFallback方法等。但其实很鸡肋，因为当前线程还是被阻塞着。

信号量模式下一种正确的配置方法，是将command的超时时间设置的尽可能长，然后在command内部的各种可能block的地方，网络IO、磁盘IO等，设置合理的超时时间。

可能更合理的做法，是hystrix去掉信号量模式下面的这个容易让人迷惑的超时时间配置。或者，是时候换一个熔断降级组件了。