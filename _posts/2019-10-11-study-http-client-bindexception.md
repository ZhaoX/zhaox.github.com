---
layout: post
title: "一次Commons-HttpClient的BindException排查"
description: "A Study of Http Client BindException"
category: Web
tags: [Web, Authentication]
---
{% include JB/setup %}

线上有个老应用，在流量增长的时候，HttpClient抛出了BindException。部分的StackTrace信息如下：

``` java
 java.net.BindException: Address already in use (Bind failed) at
 java.net.PlainSocketImpl.socketBind(Native Method) ~[?:1.8.0_162] at
 java.net.AbstractPlainSocketImpl.bind(AbstractPlainSocketImpl.java:387) ~[?:1.8.0_162] at
 java.net.Socket.bind(Socket.java:644) ~[?:1.8.0_162] at
 sun.reflect.GeneratedMethodAccessor289.invoke(Unknown Source) ~[?:?] at
 sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[?:1.8.0_162] at
 java.lang.reflect.Method.invoke(Method.java:498) ~[?:1.8.0_162] at
 org.apache.commons.httpclient.protocol.ReflectionSocketFactory.createSocket(ReflectionSocketFactory.java:139) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.protocol.DefaultProtocolSocketFactory.createSocket(DefaultProtocolSocketFactory.java:125) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.HttpConnection.open(HttpConnection.java:707) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.MultiThreadedHttpConnectionManager$HttpConnectionAdapter.open(MultiThreadedHttpConnectionManager.java:1361) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.HttpMethodDirector.executeWithRetry(HttpMethodDirector.java:387) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.HttpMethodDirector.executeMethod(HttpMethodDirector.java:171) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.HttpClient.executeMethod(HttpClient.java:397) ~[commons-httpclient-3.1.jar:?] at
 org.apache.commons.httpclient.HttpClient.executeMethod(HttpClient.java:323) ~[commons-httpclient-3.1.jar:?]`
```

### Ephemeral Port Exhausted

先Google，很多人说是操作系统的临时端口号耗尽了。倒也说得通，线上服务没有连接池，流量一大，HttpClient每创建一个连接就会占用一个临时端口号。

但我还是有疑问。

说疑问之前先简单介绍下临时端口号（Ephemeral Port）。

一个TCP连接由四元组标识：

``` C
   {source_ip, source_port, destination_ip, destination_port}
```

对于HttpClient来说，每次都是作为source创建TCP连接，也就是说destination_ip和destination_port是确定的，只需要调用系统调用connect，操作系统会自动分配source_ip和source_port。

这个分配过程不仅HttpClient的使用者不关心，HttpClient的开发者也不用关心。

不过临时端口号对操作系统来说是有限的资源，有个范围限制，同时创建的连接太多，就不够用了。再创建连接，就会报错。

比如下面这条nginx log，就是因为临时端口号耗尽，Nginx无法创建到upstream的连接了：

``` log
2016/03/18 09:08:37 [crit] 1888#1888: *13 connect() to 10.2.2.77:8081 failed (99: Cannot assign requested address) while connecting to upstream, client: 10.2.2.42, server: , request: "GET / HTTP/1.1", upstream: "http://10.2.2.77:8081/", host: "10.2.2.77"
```

这个时候我的疑问来了。

如果原因是临时端口号耗尽，HttpClient为什么会抛出BindException呢？作为创建TCP连接的source这一方，只需要系统调用connect，没必要系统调用bind啊。

如果原因是临时端口号耗尽，像上面nginx log那种错误提示才是合理的吧？

### HttpClient 3.1

猜猜猜，猜不出来，只好去看看HttpClient的代码。

老应用之老，不止年纪大，用的三方库的版本也旧。HttpClient还是commons-httpclient-3.1.jar。

``` java
package org.apache.commons.httpclient.protocol;
public final class ReflectionSocketFactory:

    public static Socket createSocket(
        final String socketfactoryName,
        final String host,
        final int port,
        final InetAddress localAddress,
        final int localPort,
        int timeout)
     throws IOException, UnknownHostException, ConnectTimeoutException
    {
        if (REFLECTION_FAILED) {
            //This is known to have failed before. Do not try it again
            return null;
        }
        // This code uses reflection to essentially do the following:
        //
        //  SocketFactory socketFactory = Class.forName(socketfactoryName).getDefault();
        //  Socket socket = socketFactory.createSocket();
        //  SocketAddress localaddr = new InetSocketAddress(localAddress, localPort);
        //  SocketAddress remoteaddr = new InetSocketAddress(host, port);
        //  socket.bind(localaddr);
        //  socket.connect(remoteaddr, timeout);
        //  return socket;
        try {
            Class socketfactoryClass = Class.forName(socketfactoryName);
            Method method = socketfactoryClass.getMethod("getDefault", 
                new Class[] {});
            Object socketfactory = method.invoke(null, 
                new Object[] {});
            method = socketfactoryClass.getMethod("createSocket", 
                new Class[] {});
            Socket socket = (Socket) method.invoke(socketfactory, new Object[] {});

            if (INETSOCKETADDRESS_CONSTRUCTOR == null) {
                Class addressClass = Class.forName("java.net.InetSocketAddress");
                INETSOCKETADDRESS_CONSTRUCTOR = addressClass.getConstructor(
                    new Class[] { InetAddress.class, Integer.TYPE });
            }

            Object remoteaddr = INETSOCKETADDRESS_CONSTRUCTOR.newInstance(
                new Object[] { InetAddress.getByName(host), new Integer(port)});

            Object localaddr = INETSOCKETADDRESS_CONSTRUCTOR.newInstance(
                    new Object[] { localAddress, new Integer(localPort)});

            if (SOCKETCONNECT_METHOD == null) {
                SOCKETCONNECT_METHOD = Socket.class.getMethod("connect", 
                    new Class[] {Class.forName("java.net.SocketAddress"), Integer.TYPE});
            }

            if (SOCKETBIND_METHOD == null) {
                SOCKETBIND_METHOD = Socket.class.getMethod("bind", 
                    new Class[] {Class.forName("java.net.SocketAddress")});
            }
            SOCKETBIND_METHOD.invoke(socket, new Object[] { localaddr});
            SOCKETCONNECT_METHOD.invoke(socket, new Object[] { remoteaddr, new Integer(timeout)});
            return socket;
        }
        catch (InvocationTargetException e) {
            Throwable cause = e.getTargetException(); 
            if (SOCKETTIMEOUTEXCEPTION_CLASS == null) {
                try {
                    SOCKETTIMEOUTEXCEPTION_CLASS = Class.forName("java.net.SocketTimeoutException");
                } catch (ClassNotFoundException ex) {
                    // At this point this should never happen. Really.
                    REFLECTION_FAILED = true;
                    return null;
                }
            }
            if (SOCKETTIMEOUTEXCEPTION_CLASS.isInstance(cause)) {
                throw new ConnectTimeoutException(
                    "The host did not accept the connection within timeout of " 
                    + timeout + " ms", cause);
            }
            if (cause instanceof IOException) {
                throw (IOException)cause;
            }
            return null;
        }
        catch (Exception e) {
            REFLECTION_FAILED = true;
            return null;
        }
    }
```

重点是这两句：

``` java
    SOCKETBIND_METHOD.invoke(socket, new Object[] { localaddr});
    SOCKETCONNECT_METHOD.invoke(socket, new Object[] { remoteaddr, new Integer(timeout)});
```

HttpClient在connect之前调用了bind，系统调用bind返回了EADDRINUSE错误：

``` C
    EADDRINUSE
        The given address is already in use.
```

然后是java.net.PlainSocketImpl.socketBind(Native Method)抛出了BindException。

这样的话，的确，是临时端口号耗尽，导致抛出了BindException，因为HttpClient在connect之前，先调用了bind。

只是，为什么要先bind呢？

### Bind before Connect

connect之前先bind，是允许的，但并没有什么好处，反而带来极大的危害。

好吧，其实在特定情况下也可能有一点好处，这里先说危害，后面再说好处。

前面说了，临时端口号是有限的资源，数量是有限制的。并且TCP连接是个四元组：

``` C
    {source_ip, source_port, destination_ip, destination_port}
```

如果我们直接调用connect，由操作系统来分配临时端口号：

``` C
    connect(socket, destination_addr, sizeof destination_addr);
```

那么操作系统就为不同的destination_ip和destination_port，分别维护临时端口号分配。

假设临时端口号数量为N，那么每一个destination_ip和destination_port的组合，都能创建N个连接。

而如果connect之前先调用bind：

``` C
    bind(socket, source_addr, sizeof source_addr);
    connect(socket, destination_addr, sizeof destination_addr);
```

那已经bind过还没释放的source_port就不会再允许bind。临时端口号就变成了不同destination之间共用的资源。

假设临时端口号数量为N，那么所有destination_ip和destination_port的组合加起来，一共只能创建N个连接。

反应到HttpClient和java应用上，举例来讲：

如果你的java应用，既要使用HttpClient访问百度，又要使用HttpClient访问Google，还要使用HttpClient访问Bing。你的操作系统临时端口号数量限制为10000。

那么直接connect，百度、Google、Bing都能同时存在10000个连接，且互相之间无影响。

先bind后connect，百度、Google、Bing加起来一共只能创建10000个连接，且互相之间有影响，需要连接百度的流量大了，连接多了超过限制了，需要连接Google和Bing的也会失败。

### HttpClient 4.4

看到这里，原因已经清楚了。接下来去找了比较新的HttpCliet版本来看是否有改进。如下是HttpClient 4.4的创建连接相关代码：

``` java
package org.apache.http.impl.pool;
public class BasicConnFactory implements ConnFactory<HttpHost, HttpClientConnection>:

    @Override
    public HttpClientConnection create(final HttpHost host) throws IOException {
        final String scheme = host.getSchemeName();
        Socket socket = null;
        if ("http".equalsIgnoreCase(scheme)) {
            socket = this.plainfactory != null ? this.plainfactory.createSocket() :
                    new Socket();
        } if ("https".equalsIgnoreCase(scheme)) {
            socket = (this.sslfactory != null ? this.sslfactory :
                    SSLSocketFactory.getDefault()).createSocket();
        }
        if (socket == null) {
            throw new IOException(scheme + " scheme is not supported");
        }
        final String hostname = host.getHostName();
        int port = host.getPort();
        if (port == -1) {
            if (host.getSchemeName().equalsIgnoreCase("http")) {
                port = 80;
            } else if (host.getSchemeName().equalsIgnoreCase("https")) {
                port = 443;
            }
        }
        socket.setSoTimeout(this.sconfig.getSoTimeout());
        if (this.sconfig.getSndBufSize() > 0) {
            socket.setSendBufferSize(this.sconfig.getSndBufSize());
        }
        if (this.sconfig.getRcvBufSize() > 0) {
            socket.setReceiveBufferSize(this.sconfig.getRcvBufSize());
        }
        socket.setTcpNoDelay(this.sconfig.isTcpNoDelay());
        final int linger = this.sconfig.getSoLinger();
        if (linger >= 0) {
            socket.setSoLinger(true, linger);
        }
        socket.setKeepAlive(this.sconfig.isSoKeepAlive());
        socket.connect(new InetSocketAddress(hostname, port), this.connectTimeout);
        return this.connFactory.createConnection(socket);
    }
```

果然，改掉了，没有在connect之前先bind了。直接调用的connect：

``` java
    socket.connect(new InetSocketAddress(hostname, port), this.connectTimeout);
```

有条件还是要积极升级各种库的版本啊。

### 连接池、熔断降级

像这次这个老应用这种，对三方依赖占用的资源没有限制，也没有熔断降级。确实还是太粗放了。

首先连接池必须有，连接复用提升效率，并且可以限制连接数，对客户端对服务端都好。HttpClient本身就支持连接池。

另外对三方依赖要有熔断降级，当一个依赖方出现问题或者相关流量大的时候，该降级降级，该熔断熔断，尽量的将影响控制到最小范围。熔断降级可以用hystrix。

### Linux Ephemeral Port Range

就着这次问题排查，总结下临时端口号相关知识。因为每个操作系统不同，这里主要介绍linux。

临时端口号范围：

``` bash
# sysctl net.ipv4.ip_local_port_range
net.ipv4.ip_local_port_range = 32768   61000
```

假设我们的业务逻辑处理非常快网络也好，一个连接从建立到关闭在1ms内，那么一个临时端口号被分配到下次可以使用，只需要等待TCP连接的TIME_WAIT状态结束即可。

TIME_WAIT状态的持续时间定义内核代码$KERNEL/include/net/tcp.h中：

``` C
#define TCP_TIMEWAIT_LEN (60*HZ)
```

以上皆为多数linux内核的默认值。

可以看到，默认临时端口号共有61000-32768=28232个。一个端口号被使用后，最少需要60秒才能释放。

也就是说，如果固定了source_ip、destination_ip、destination_port，每分钟最多只能创建28232个连接，平均每秒(61000-32768)/60=470.5个。

几百个，一个非常小的数值。对于流量大的业务，很容易出问题。更何况上面HttpClient先bind再connect。

如果想要改变这种情况，提高能够同时创建的连接数量。有以下几种办法：

- 调大net.ipv4.ip_local_port_range

这个范围可以调大，但最大不能超过65536，最小不能超过1234

比如可以调成这样：

``` bash
sysctl net.ipv4.ip_local_port_range="1235 65000"
```

这个操作没什么风险，可以适当调大。

- 允许端口快速复用

也就是允许还处在TIME_WAIT状态的TCP连接占用的本地端口，被其它TCP连接使用。系统默认是不允许的。

可以在系统层面配置net.ipv4.tcp_tw_reuse：

``` bash
sysctl net.ipv4.tcp_tw_reuse=1
```

也可以为特定的socket设置SO_REUSEADDR选项。

不过TIME_WAIT状态本身是有意义的，用来保证TCP连接的可靠性。允许复用TIME_WAIT状态的连接占用的端口号，虽然资源利用率提供，但也可能带来难以排查和解决的隐藏问题，需要慎重开启相关配置。

诚如man ip(7)所述：

> A TCP local socket address that has been bound is unavailable for some time after closing, unless the SO_REUSEADDR flag has been set. Care should be taken when using this flag as it makes TCP less reliable.

- 使用多个source_ip

这个方案比较tricky，如前所述，固定了source_ip、destination_ip、destination_port，临时端口号数量固定。

如果有多个source_ip，那么可用的临时端口号数量可以成倍增长。

怎么用呢，需要利用系统调用bind的一个特性。如果在bind的时候，指定source_ip，但source_port设置为0，并且为socket设置IP_BIND_ADDRESS_NO_PORT选项。

> tcp sockets before binding to a specific source ip with port 0 if you're going to use the socket for connect() rather then listen() this allows the kernel to delay allocating the source port until connect() time at which point it is much cheaper

这样在bind的时候，系统不会分配端口号，而是等到connect时再分配，但又指定了source_ip。

想要用这个方案，就必须先bind再connect了。这就是前文所述，bind before connect有可能的好处。

这个方案不实用，大部分情况下，服务器只有一个可用ip，这个方案都是用不了的。即便能用，用起来也比较麻烦。

### Reference

[https://idea.popcount.org/2014-04-03-bind-before-connect/](https://idea.popcount.org/2014-04-03-bind-before-connect/)
[https://www.nginx.com/blog/overcoming-ephemeral-port-exhaustion-nginx-plus/](https://www.nginx.com/blog/overcoming-ephemeral-port-exhaustion-nginx-plus/)
[https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
[https://github.com/torvalds/linux/blob/4ba9920e5e9c0e16b5ed24292d45322907bb9035/net/ipv4/inet_connection_sock.c#L118](https://github.com/torvalds/linux/blob/4ba9920e5e9c0e16b5ed24292d45322907bb9035/net/ipv4/inet_connection_sock.c#L118)