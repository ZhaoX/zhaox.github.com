---
layout: post
title: "isDebugEnabled有什么用？"
description: ""
category: Java
tags: [Java, Log]
---
{% include JB/setup %}

这几天在读Spring MVC源码时，发现了如下代码：

``` java
if (logger.isDebugEnabled()) {
    logger.debug("Using ThemeResolver [" + this.themeResolver + "]");
}
```

初看有些不解，觉着在设置了高于debug级别的log输出时，logger.debug(...)原本就不会输出，为什么还需要logger.isDebugEnabled()。

在org.apache.commons.logging.Log的代码注释中找到了答案：

``` java
 * <p>Performance is often a logging concern.
 * By examining the appropriate property,
 * a component can avoid expensive operations (producing information
 * to be logged).</p>
 *
 * <p> For example,
 * <code><pre>
 *    if (log.isDebugEnabled()) {
 *        ... do something expensive ...
 *        log.debug(theResult);
 *    }
 * </pre></code>
 * </p>
```

也就是说，是为了避免一些logger.debug(...)准备工作的执行，从而提高性能。至于本文提到的例子，应该属于最简单的情况，避免无用的String参数拼接。

相应的，也不是只有debug级别才有这个方法，每一个log级别都有对应的enabled方法供调用。org.apache.commons.logging.Log接口的代码如下：
``` java
public interface Log {

    // ----------------------------------------------------- Logging Properties

    public boolean isDebugEnabled();

    public boolean isErrorEnabled();

    public boolean isFatalEnabled();

    public boolean isInfoEnabled();

    public boolean isTraceEnabled();

    public boolean isWarnEnabled();

    // -------------------------------------------------------- Logging Methods

    public void trace(Object message);

    public void trace(Object message, Throwable t);

    public void debug(Object message);

    public void debug(Object message, Throwable t);

    public void info(Object message);

    public void info(Object message, Throwable t);

    public void warn(Object message);

    public void warn(Object message, Throwable t);

    public void error(Object message);

    public void error(Object message, Throwable t);

    public void fatal(Object message);

    public void fatal(Object message, Throwable t);

}
```
