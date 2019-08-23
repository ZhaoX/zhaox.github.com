---
layout: post
title: "危险的Hystrix线程池"
description: "Hystrix Thread Pool"
category: Java
tags: [Java, Hystrix]
---
{% include JB/setup %}

本文介绍Hystrix线程池的工作原理和参数配置，指出存在的问题并提供规避方案，阅读本文需要对Hystrix有一定的了解。

文本讨论的内容，基于hystrix 1.5.18：

``` xml
    <dependency>
      <groupId>com.netflix.hystrix</groupId>
      <artifactId>hystrix-core</artifactId>
      <version>1.5.18</version>
    </dependency>
```

### 线程池和Hystrix Command之间的关系

当hystrix command的隔离策略配置为线程，也就是execution.isolation.strategy设置为THREAD时，command中的代码会放到线程池里执行，跟发起command调用的线程隔离开。摘要官方wiki如下：

> execution.isolation.strategy
> 
> This property indicates which isolation strategy HystrixCommand.run() executes with, one of the following two choices:
> 
> THREAD — it executes on a separate thread and concurrent requests are limited by the number of threads in the thread-pool
> SEMAPHORE — it executes on the calling thread and concurrent requests are limited by the semaphore count

一个线上的服务，往往会有很多hystrix command分别用来管理不同的外部依赖。 但会有几个hystrix线程池存在呢，这些command跟线程池的对应关系又是怎样的呢，是一对一吗？

答案是不一定，command跟线程池可以做到一对一，但通常不是，受到HystrixThreadPoolKey和HystrixCommandGroupKey这两项配置的影响。

优先采用HystrixThreadPoolKey来标识线程池，如果没有配置HystrixThreadPoolKey那么就使用HystrixCommandGroupKey来标识。command跟线程池的对应关系，就看HystrixCommandKey、HystrixThreadPoolKey、HystrixCommandGroupKey这三个参数的配置。

获取线程池标识的代码如下，可以看到跟我的描述是一致的：

``` java
    /*
     * ThreadPoolKey
     *
     * This defines which thread-pool this command should run on.
     *
     * It uses the HystrixThreadPoolKey if provided, then defaults to use HystrixCommandGroup.
     *
     * It can then be overridden by a property if defined so it can be changed at runtime.
     */
    private static HystrixThreadPoolKey initThreadPoolKey(HystrixThreadPoolKey threadPoolKey, HystrixCommandGroupKey groupKey, String threadPoolKeyOverride) {
        if (threadPoolKeyOverride == null) {
            // we don't have a property overriding the value so use either HystrixThreadPoolKey or HystrixCommandGroup
            if (threadPoolKey == null) {
                /* use HystrixCommandGroup if HystrixThreadPoolKey is null */
                return HystrixThreadPoolKey.Factory.asKey(groupKey.name());
            } else {
                return threadPoolKey;
            }
        } else {
            // we have a property defining the thread-pool so use it instead
            return HystrixThreadPoolKey.Factory.asKey(threadPoolKeyOverride);
        }
    }
```

Hystrix会保证同一个线程池标识只会创建一个线程池：

``` java
    /*
     * Use the String from HystrixThreadPoolKey.name() instead of the HystrixThreadPoolKey instance as it's just an interface and we can't ensure the object
     * we receive implements hashcode/equals correctly and do not want the default hashcode/equals which would create a new threadpool for every object we get even if the name is the same
     */
    /* package */final static ConcurrentHashMap<String, HystrixThreadPool> threadPools = new ConcurrentHashMap<String, HystrixThreadPool>();

    /**
     * Get the {@link HystrixThreadPool} instance for a given {@link HystrixThreadPoolKey}.
     * <p>
     * This is thread-safe and ensures only 1 {@link HystrixThreadPool} per {@link HystrixThreadPoolKey}.
     *
     * @return {@link HystrixThreadPool} instance
     */
    /* package */static HystrixThreadPool getInstance(HystrixThreadPoolKey threadPoolKey, HystrixThreadPoolProperties.Setter propertiesBuilder) {
        // get the key to use instead of using the object itself so that if people forget to implement equals/hashcode things will still work
        String key = threadPoolKey.name();

        // this should find it for all but the first time
        HystrixThreadPool previouslyCached = threadPools.get(key);
        if (previouslyCached != null) {
            return previouslyCached;
        }

        // if we get here this is the first time so we need to initialize
        synchronized (HystrixThreadPool.class) {
            if (!threadPools.containsKey(key)) {
                threadPools.put(key, new HystrixThreadPoolDefault(threadPoolKey, propertiesBuilder));
            }
        }
        return threadPools.get(key);
    }
```

### Hystrix线程池参数一览

- [coreSize](https://github.com/Netflix/Hystrix/wiki/Configuration#coreSize) 核心线程数量
- [maximumSize](https://github.com/Netflix/Hystrix/wiki/Configuration#maximumSize) 最大线程数量
- [allowMaximumSizeToDivergeFromCoreSize](https://github.com/Netflix/Hystrix/wiki/Configuration#allowMaximumSizeToDivergeFromCoreSize) 允许maximumSize大于coreSize，只有配了这个值coreSize才有意义
- [keepAliveTimeMinutes](https://github.com/Netflix/Hystrix/wiki/Configuration#keepAliveTimeMinutes) 超过这个时间多于coreSize数量的线程会被回收，只有maximumsize大于coreSize，这个值才有意义
- [maxQueueSize](https://github.com/Netflix/Hystrix/wiki/Configuration#maxQueueSize) 任务队列的最大大小，当线程池的线程线程都在工作，也不能创建新的线程的时候，新的任务会进到队列里等待
- [queueSizeRejectionThreshold](https://github.com/Netflix/Hystrix/wiki/Configuration#queueSizeRejectionThreshold) 任务队列中存储的任务数量超过这个值，线程池拒绝新的任务。这跟maxQueueSize本来是一回事，只是受限于hystrix的实现方式maxQueueSize不能动态配置，所以有了这个配置。

### 根据给定的线程池参数猜测线程池表现

可以看到hystrix的线程池参数跟JDK线程池ThreadPoolExecutor参数很像但又不一样，即便是完整地看了文档，仍然让人迷惑。不过无妨，先来猜猜几种配置下的表现。

``` json
coreSize = 2; maxQueueSize = 10
```

线程池中常驻2个线程。新任务提交到线程池，有空闲线程则直接执行，否则入队等候。等待队列中的任务数=10时，拒绝接受新任务。

``` json
coreSize = 2; maximumSize = 5; maxQueueSize = -1
```

线程池中常驻2个线程。新任务提交到线程池，有空闲线程则直接执行，没有空闲线程时，如果当前线程数小于5则创建1个新的线程用来执行任务，否则拒绝任务。

``` json
coreSize = 2; maximumSize = 5; maxQueueSize = 10
```

这种配置下从官方文档中已经看不出来实际表现会是怎样的。猜测有如下两种可能：

- 可能一。线程池中常驻2个线程。新任务提交到线程池，2个线程中有空闲则直接执行，否则入队等候。当2个线程都在工作且等待队列中的任务数=10时，开始为新任务创建线程，直到线程数量为5，此时开始拒绝新任务。这样的话，对资源敏感型的任务比较友好，这也是JDK线程池ThreadPoolExecutor的行为。

- 可能二。线程池中常驻2个线程。新任务提交到线程池，有空闲线程则直接执行，没有空闲线程时，如果当前线程数小于5则创建1个新的线程用来执行任务。当线程数量达到5个且都在工作时，任务入队等候。等待队列中的任务数=10时，拒绝接受新任务。这样的话，对延迟敏感型的任务比较友好。

两种情况都有可能，从文档中无法确定究竟如何。

### 并发情况下Hystrix线程池的真正表现

本节中，通过测试来看看线程池的行为究竟会怎样。

还是这个配置：

``` json
coreSize = 2; maximumSize = 5; maxQueueSize = 10
```

我们通过不断提交任务到hystrix线程池，并且在任务的执行代码中使用CountDownLatch占住线程来模拟测试，代码如下：

``` java
public class HystrixThreadPoolTest {

  public static void main(String[] args) throws InterruptedException {
    final int coreSize = 2, maximumSize = 5, maxQueueSize = 10;
    final String commandName = "TestThreadPoolCommand";

    final HystrixCommand.Setter commandConfig = HystrixCommand.Setter
        .withGroupKey(HystrixCommandGroupKey.Factory.asKey(commandName))
        .andCommandKey(HystrixCommandKey.Factory.asKey(commandName))
        .andCommandPropertiesDefaults(
            HystrixCommandProperties.Setter()
                .withExecutionTimeoutEnabled(false))
        .andThreadPoolPropertiesDefaults(
            HystrixThreadPoolProperties.Setter()
                .withCoreSize(coreSize)
                .withMaximumSize(maximumSize)
                .withAllowMaximumSizeToDivergeFromCoreSize(true)
                .withMaxQueueSize(maxQueueSize)
                .withQueueSizeRejectionThreshold(maxQueueSize));

    // Run command once, so we can get metrics.
    HystrixCommand<Void> command = new HystrixCommand<Void>(commandConfig) {
      @Override protected Void run() throws Exception {
        return null;
      }
    };
    command.execute();
    Thread.sleep(100);

    final CountDownLatch stopLatch = new CountDownLatch(1);
    List<Thread> threads = new ArrayList<Thread>();

    for (int i = 0; i < coreSize + maximumSize + maxQueueSize; i++) {
      final int fi = i + 1;

      Thread thread = new Thread(new Runnable() {
        public void run() {
          try {
            HystrixCommand<Void> command = new HystrixCommand<Void>(commandConfig) {
              @Override protected Void run() throws Exception {
                stopLatch.await();
                return null;
              }
            };
            command.execute();
          } catch (HystrixRuntimeException e) {
            System.out.println("Started Jobs: " + fi);
            System.out.println("Job:" + fi + " got rejected.");
            printThreadPoolStatus();
            System.out.println();
          }
        }
      });
      threads.add(thread);
      thread.start();
      Thread.sleep(200);

      if(fi == coreSize || fi == coreSize + maximumSize || fi == coreSize + maxQueueSize ) {
        System.out.println("Started Jobs: " + fi);
        printThreadPoolStatus();
        System.out.println();
      }
    }

    stopLatch.countDown();

    for (Thread thread : threads) {
      thread.join();
    }

  }

  static void printThreadPoolStatus() {
    for (HystrixThreadPoolMetrics threadPoolMetrics : HystrixThreadPoolMetrics.getInstances()) {
      String name = threadPoolMetrics.getThreadPoolKey().name();
      Number poolSize = threadPoolMetrics.getCurrentPoolSize();
      Number queueSize = threadPoolMetrics.getCurrentQueueSize();
      System.out.println("ThreadPoolKey: " + name + ", PoolSize: " + poolSize + ", QueueSize: " + queueSize);
    }

  }

}
```

执行代码得到如下输出：

``` java 
// 任务数 = coreSize。此时coreSize个线程在工作
Started Jobs: 2
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 0

// 任务数 > coreSize。此时仍然只有coreSize个线程，多于coreSize的任务进入等候队列，没有创建新的线程  
Started Jobs: 7
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 5

// 任务数 = coreSize + maxQueueSize。此时仍然只有coreSize个线程，多于coreSize的任务进入等候队列，没有创建新的线程  
Started Jobs: 12
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 10

// 任务数 > coreSize + maxQueueSize。此时仍然只有coreSize个线程，等候队列已满，新增任务被拒绝 
Started Jobs: 13
Job:13 got rejected.
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 10

Started Jobs: 14
Job:14 got rejected.
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 10

Started Jobs: 15
Job:15 got rejected.
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 10

Started Jobs: 16
Job:16 got rejected.
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 10

Started Jobs: 17
Job:17 got rejected.
ThreadPoolKey: TestThreadPoolCommand, PoolSize: 2, QueueSize: 10
```

完整的测试代码，参见[这里](https://github.com/ZhaoX/java-experiment/blob/master/hystrix/src/main/java/HystrixThreadPoolTest.java)

可以看到Hystrix线程池的实际表现，跟之前的两种猜测都不同，跟JDK线程池的表现不同，跟另一种合理猜测也不通。当maxSize > coreSize && maxQueueSize != -1的时候，maxSize这个参数根本就不起作用，线程数量永远不会超过coreSize，对于的任务入队等候，队列满了，就直接拒绝新任务。

不得不说，这是一种让人疑惑的，非常危险的，容易配置错误的线程池表现。

### JDK线程池ThreadPoolExecutor

继续分析Hystrix线程池的原理之前，先来复习一下JDK中的线程池。

只说跟本文讨论的内容相关的参数：

- corePoolSize核心线程数，maximumPoolSize最大线程数。这个两个参数跟hystrix线程池的coreSize和maximumSize含义是一致的。
- workQueue任务等候队列。跟hystrix不同，jdk线程池的等候队列不是指定大小，而是需要使用方提供一个BlockingQueue。
- handler当线程池无法接受任务时的处理器。hystrix是直接拒绝，jdk线程池可以定制。

可以看到，jdk的线程池使用起来更加灵活。配置参数的含义也十分清晰，没有hystrx线程池里面allowMaximumSizeToDivergeFromCoreSize、queueSizeRejectionThreshold这种奇奇怪怪让人疑惑的参数。

关于jdk线程池的参数配置，参加如下jdk源码：

``` java

    /**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

那么在跟hystrix线程池对应的参数配置下，jdk线程池的表现会怎样呢？

``` json
corePoolSize = 2; maximumPoolSize = 5; workQueue = new ArrayBlockingQueue(10); handler = new ThreadPoolExecutor.DiscardPolicy()
```

这里不再测试了，直接给出答案。线程池中常驻2个线程。新任务提交到线程池，2个线程中有空闲则直接执行，否则入队等候。当2个线程都在工作且等待队列中的任务数=10时，开始为新任务创建线程，直到线程数量为5，此时开始拒绝新任务。

相关逻辑涉及的源码贴在下面。值得一提的是，jdk线程池并不根据等候任务的数量来判断等候队列是否已满，而是直接调用workQueue的offer方法，如果workQueue接受了那就入队等候，否则执行拒绝策略。

``` java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

可以看到hystrix线程池的配置参数跟jdk线程池是非常像的，从名字到含义，都基本一致。

### 为什么

事实上hystrix的线程池，就是在jdk线程池的基础上实现的。相关代码如下：

``` java

    public ThreadPoolExecutor getThreadPool(final HystrixThreadPoolKey threadPoolKey, HystrixThreadPoolProperties threadPoolProperties) {
        final ThreadFactory threadFactory = getThreadFactory(threadPoolKey);

        final boolean allowMaximumSizeToDivergeFromCoreSize = threadPoolProperties.getAllowMaximumSizeToDivergeFromCoreSize().get();
        final int dynamicCoreSize = threadPoolProperties.coreSize().get();
        final int keepAliveTime = threadPoolProperties.keepAliveTimeMinutes().get();
        final int maxQueueSize = threadPoolProperties.maxQueueSize().get();
        final BlockingQueue<Runnable> workQueue = getBlockingQueue(maxQueueSize);

        if (allowMaximumSizeToDivergeFromCoreSize) {
            final int dynamicMaximumSize = threadPoolProperties.maximumSize().get();
            if (dynamicCoreSize > dynamicMaximumSize) {
                logger.error("Hystrix ThreadPool configuration at startup for : " + threadPoolKey.name() + " is trying to set coreSize = " +
                        dynamicCoreSize + " and maximumSize = " + dynamicMaximumSize + ".  Maximum size will be set to " +
                        dynamicCoreSize + ", the coreSize value, since it must be equal to or greater than the coreSize value");
                return new ThreadPoolExecutor(dynamicCoreSize, dynamicCoreSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
            } else {
                return new ThreadPoolExecutor(dynamicCoreSize, dynamicMaximumSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
            }
        } else {
            return new ThreadPoolExecutor(dynamicCoreSize, dynamicCoreSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory);
        }
    }

    public BlockingQueue<Runnable> getBlockingQueue(int maxQueueSize) {
        /*
         * We are using SynchronousQueue if maxQueueSize <= 0 (meaning a queue is not wanted).
         * <p>
         * SynchronousQueue will do a handoff from calling thread to worker thread and not allow queuing which is what we want.
         * <p>
         * Queuing results in added latency and would only occur when the thread-pool is full at which point there are latency issues
         * and rejecting is the preferred solution.
         */
        if (maxQueueSize <= 0) {
            return new SynchronousQueue<Runnable>();
        } else {
            return new LinkedBlockingQueue<Runnable>(maxQueueSize);
        }
    }

```

既然hystrix线程池基于jdk线程池实现，为什么在如下两个基本一致的配置上，行为却不一样呢？

``` json
//hystrix
coreSize = 2; maximumSize = 5; maxQueueSize = 10

//jdk
corePoolSize = 2; maximumPoolSize = 5; workQueue = new ArrayBlockingQueue(10); handler = new ThreadPoolExecutor.DiscardPolicy()
```

jdk在队列满了之后会创建线程执行新任务直到线程数量达到maximumPoolSize，而hystrix在队列满了之后直接拒绝新任务，maximumSize这项配置成了摆设。

原因就在于hystrix判断队列是否满是否要拒绝新任务，没有通过jdk线程池在判断，而是自己判断的。参见如下hystrix源码：

``` java
    public boolean isQueueSpaceAvailable() {
        if (queueSize <= 0) {
            // we don't have a queue so we won't look for space but instead
            // let the thread-pool reject or not
            return true;
        } else {
            return threadPool.getQueue().size() < properties.queueSizeRejectionThreshold().get();
        }
    }

    public Subscription schedule(Action0 action, long delayTime, TimeUnit unit) {
        if (threadPool != null) {
            if (!threadPool.isQueueSpaceAvailable()) {
                throw new RejectedExecutionException("Rejected command because thread-pool queueSize is at rejection threshold.");
            }
        }
        return worker.schedule(new HystrixContexSchedulerAction(concurrencyStrategy, action), delayTime, unit);
    }
```

可以看到hystrix在队列大小达到maxQueueSize时，根本不会往底层的ThreadPoolExecutor提交任务。ThreadPoolExecutor也就没有机会判断workQueue能不能offer，更不能创建新的线程了。

### 怎么办

对用惯了jdk的ThreadPoolExecutor的人来说，再用hystrix的确容易出错，笔者就曾在多个重要线上服务的代码里看到过错误的配置，称一声危险的hystrix线程池不为过。

那怎么办呢？

#### 配置的时候规避问题

同时配置maximumSize > coreSize，maxQueueSize > 0，像下面这样，是不行了。

``` json
coreSize = 2; maximumSize = 5; maxQueueSize = 10
```

妥协一下，如果对延迟比较看重，配置maximumSize > coreSize，maxQueueSize = -1。这样在任务多的时候，不会有等候队列，直接创建新线程执行任务。

``` java
coreSize = 2; maximumSize = 5; maxQueueSize = -1
```

如果对资源比较看重, 不希望创建过多线程，配置maximumSize = coreSize，maxQueueSize > 0。这样在任务多的时候，会进等候队列，直到有线程空闲或者超时。

``` java
coreSize = 2; maximumSize = 2; maxQueueSize = 10
```

#### 在hystrix上修复这个问题

技术上是可行的，有很多方案可以做到。但Netflix已经宣布不再维护hystrix了，这条路也就不通了，除非维护自己的hystrix分支版本。

### Reference
https://github.com/Netflix/Hystrix/wiki/Configuration
https://github.com/Netflix/Hystrix/issues/1589
https://github.com/Netflix/Hystrix/pull/1670
