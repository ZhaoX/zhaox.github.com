---
layout: post
title: "危险的Hystrix线程池"
description: "Hystrix Thread Pool"
category: Java
tags: [Java, Hystrix]
---
{% include JB/setup %}

本文介绍Hystrix线程池的工作原理和参数配置，指出存在的问题并提供规避方案。注意本文不是Hystrix的入门读物，阅读本文需要对Hystrix有一定的了解。

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

可以看到hystrix的线程池参数跟JDK线程池ThreadPoolExecutor参数很像但又不一样，即便是完整地看了文档，仍然让人迷惑。不过无妨，先来猜猜看。

### JDK线程池ThreadPoolExecutor

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

### 并发情况下Hystrix线程池的真正表现

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

```

### 为什么

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


### 怎么办

配置的时候规避问题

自定义

### Reference
https://github.com/Netflix/Hystrix/wiki/Configuration
https://github.com/Netflix/Hystrix/issues/1589
https://github.com/Netflix/Hystrix/pull/1670
