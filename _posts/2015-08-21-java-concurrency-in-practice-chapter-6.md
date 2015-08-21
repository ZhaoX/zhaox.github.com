---
layout: post
title: "读Java Concurrency in Practice. 第六章."
description: ""
category:java
tags: [Java]
---
{% include JB/setup %}

这一章开讲任务执行。绝大多数并发程序的工作都可以分解为抽象的、互不相关的工作单元，称之为任务（Task）。

###使用java线程来执行任务
以web服务器的实现举例, 此时将用户的一次连接，当做一个独立的任务。

1. 单线程顺序执行所有任务。

```java
ServerSocket socket = new ServerSocket(80);
while(true) {
    Socket connection = socket.accept();
    handleRequest(connection);
}
```
这是最简单的方式，但效率也是最低的。通常在处理单用户的批量任务的时候，才适用。


2. 为每一个任务启动一个线程来执行。

```java
ServerSocket socket = new ServerSocket(80);
while(true) {
    final Socket connection = socket.accept();
    Runnable task = new Runnable() {
        public void run() {
            handleRequest(connection);
	}
    }
    new Thread(task).start();
}
```
这种方式的线程数没有限制，没有限制的线程数有如下几个缺点。a.每一个任务都要创建和销毁线程，带来额外开销；b.当线程数过多时，会占用过多的内存，会导致CPU大量的上下文切换；c.当线程数超过系统限制时，会导致系统崩溃，通常的结果是OutOfMemoryError。


###java的Executor Framework
Executor Framework提供了标准的方式，将任务的描述提交和任务的执行解耦。在同样的任务描述和提交的方式下，通过采用不同的executor实现类，可以实现不同的执行策略，单线程顺序执行、线程池执行等等。 Executor可以方便用来实现生产者消费者模式，其中，提交任务的线程可以看做是生产者（生产需要被执行的任务），而执行任务的线程则可以看做是消费者。


1. 使用Executor实现web服务器。

//不同的Executor，可以实现不同的执行策略，但任务提交方式是相同的。

```java
Executor exec = ...;
ServerSocket socket = new ServerSocket(80);
while(true) {
    final Socket connection = socket.accept();
    Runnable task = new Runnable() {
        public void run() {
            handleRequest(connection);
	}
    }
    exec.execute(task);
}
```


2. 什么是执行策略。

    * 任务会在哪个线程中被执行？
    * 任务会按照什么顺序被执行？FIFO，LIFO，按照优先级？
    * 多少个任务被同时执行？
    * 允许多少个任务缓存在队列中等待被执行？
    * 如果系统负载过重，哪些任务应该被放弃执行，又应该怎样通知应用程序？
    * 在执行任务之前和任务执行之后，应该做什么样的操作？

3. 需要管理Executor的生命周期怎么办, 比如想要关闭Executor？答：使用ExecutorService。


4. 执行调度任务，可以使用ScheduledThreadPoolExecutor。与Timer相比，它更优秀。比如a.ScheduledThreadPoolExecutor可以使用多个线程执行任务；b.可以更好地处理执行任务中遇到异常的情况。


5. 需要任务的执行结果怎么办？使用Callable和Future。


6. 提交了一系列的任务，希望在这些任务有执行结果的时候就立刻获取结果？用Future的get方法可以做到，但是使用ExecutorCompletionService可以做得更好, ExecutorCompletionService。ExecutorCompletionService维护一个BlockingQueue用来保存已经执行完成的任务，并通过实现FutureTask的子类，在任务结束时将任务加入到该队列中。这样一来，又是一个生产者消费者模型，执行任务的ExecutorCompletionService可以看做是生产者，而处理已经完成任务的线程可以看做是消费者。

```java
private class QueueingFuture extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task) {
        super(task, null);
        this.task = task;
    }
    protected void done() { completionQueue.add(task); }
    private final Future<V> task;
}
```


7. 需要批量提交任务，并获取这些任务的执行结果？使用ExecutorService接口的invokeAll方法, 但需要注意的是, 在AbstractExecutorService的实现中，这个方法忽略了任务执行过程中抛出的异常。


```java
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks) {
            RunnableFuture<T> f = newTaskFor(t);
            futures.add(f);
            execute(f);
        }
        for (Future<T> f : futures) {
            if (!f.isDone()) {
                try {
                    f.get();
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                }
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            for (Future<T> f : futures)
                f.cancel(true);
    }
}
```


invokeAll方法还有一个指定超时时间的版本，其实现方式是通过在执行过程中不断减时间。


```java
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                     long timeout, TimeUnit unit)
    throws InterruptedException {
    if (tasks == null || unit == null)
        throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks)
            futures.add(newTaskFor(t));

        long lastTime = System.nanoTime();

        // Interleave time checks and calls to execute in case
        // executor doesn't have any/much parallelism.
        Iterator<Future<T>> it = futures.iterator();
        while (it.hasNext()) {
            execute((Runnable)(it.next()));
            long now = System.nanoTime();
            nanos -= now - lastTime;
            lastTime = now;
            if (nanos <= 0)
                return futures;
        }

        for (Future<T> f : futures) {
            if (!f.isDone()) {
                if (nanos <= 0)
                    return futures;
                try {
                    f.get(nanos, TimeUnit.NANOSECONDS);
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                } catch (TimeoutException toe) {
                    return futures;
                }
                long now = System.nanoTime();
                nanos -= now - lastTime;
                lastTime = now;
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            for (Future<T> f : futures)
                f.cancel(true);
    }
}
```


8. 需要批量提交任务，并且只要这些任务中有一个执行完成就获取结果？使用ExecutorService接口的invokeAny方法, 事实上，在AbstractExecutorService的实现中，invokeAny就是使用ExecutorCompleteService实现的。

```java
private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                        boolean timed, long nanos)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (tasks == null)
        throw new NullPointerException();
    int ntasks = tasks.size();
    if (ntasks == 0)
        throw new IllegalArgumentException();
    List<Future<T>> futures= new ArrayList<Future<T>>(ntasks);
    ExecutorCompletionService<T> ecs =
        new ExecutorCompletionService<T>(this);

    // For efficiency, especially in executors with limited
    // parallelism, check to see if previously submitted tasks are
    // done before submitting more of them. This interleaving
    // plus the exception mechanics account for messiness of main
    // loop.

    try {
        // Record exceptions so that if we fail to obtain any
        // result, we can throw the last exception we got.
        ExecutionException ee = null;
        long lastTime = timed ? System.nanoTime() : 0;
        Iterator<? extends Callable<T>> it = tasks.iterator();

        // Start one task for sure; the rest incrementally
        futures.add(ecs.submit(it.next()));
        --ntasks;
        int active = 1;

        for (;;) {
            Future<T> f = ecs.poll();
            if (f == null) {
                if (ntasks > 0) {
                    --ntasks;
                    futures.add(ecs.submit(it.next()));
                    ++active;
                }
                else if (active == 0)
                    break;
                else if (timed) {
                    f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                    if (f == null)
                        throw new TimeoutException();
                    long now = System.nanoTime();
                    nanos -= now - lastTime;
                    lastTime = now;
                }
                else
                    f = ecs.take();
            }
            if (f != null) {
                --active;
                try {
                    return f.get();
                } catch (ExecutionException eex) {
                    ee = eex;
                } catch (RuntimeException rex) {
                    ee = new ExecutionException(rex);
                }
            }
        }

        if (ee == null)
            ee = new ExecutionException();
        throw ee;

    } finally {
        for (Future<T> f : futures)
            f.cancel(true);
    }
}
```

