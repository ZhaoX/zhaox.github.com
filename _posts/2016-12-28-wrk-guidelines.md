---
layout: post
title: "Http压测工具wrk使用指南"
description: ""
category: Benchmark 
tags: [Http, Web, WRK]
---
{% include JB/setup %}

用过了很多压测工具，却一直没找到中意的那款。最近试了wrk感觉不错，写下这份使用指南给自己备忘用，如果能帮到你，那也很好。

### 安装

wrk支持大多数类UNIX系统，不支持windows。需要操作系统支持LuaJIT和OpenSSL，不过不用担心，大多数类Unix系统都支持。安装wrk非常简单，只要从github上下载wrk源码，在项目路径下执行make命令即可。

```
git clone https://github.com/wg/wrk

make
```
make之后，会在项目路径下生成可执行文件wrk，随后就可以用其进行HTTP压测了。可以把这个可执行文件拷贝到某个已在path中的路径，比如/usr/local/bin，这样就可以在任何路径直接使用wrk了。

默认情况下wrk会使用自带的LuaJIT和OpenSSL，如果你想使用系统已安装的版本，可以使用WITH_LUAJIT和WITH_OPENSSL这两个选项来指定它们的路径。比如：

```
make WITH_LUAJIT=/usr WITH_OPENSSL=/usr
```

### 基本使用

1. 命令行敲下wrk，可以看到使用帮助

```
Usage: wrk <options> <url>                            
  Options:                                            
    -c, --connections <N>  Connections to keep open   
    -d, --duration    <T>  Duration of test           
    -t, --threads     <N>  Number of threads to use   
                                                      
    -s, --script      <S>  Load Lua script file       
    -H, --header      <H>  Add header to request      
        --latency          Print latency statistics   
        --timeout     <T>  Socket/request timeout     
    -v, --version          Print version details      
                                                      
  Numeric arguments may include a SI unit (1k, 1M, 1G)
  Time arguments may include a time unit (2s, 2m, 2h)
```
简单翻成中文：

```
使用方法: wrk <选项> <被测HTTP服务的URL>                            
  Options:                                            
    -c, --connections <N>  跟服务器建立并保持的TCP连接数量  
    -d, --duration    <T>  压测时间           
    -t, --threads     <N>  使用多少个线程进行压测   
                                                      
    -s, --script      <S>  指定Lua脚本路径       
    -H, --header      <H>  为每一个HTTP请求添加HTTP头      
        --latency          在压测结束后，打印延迟统计信息   
        --timeout     <T>  超时时间     
    -v, --version          打印正在使用的wrk的详细版本信息
                                                      
  <N>代表数字参数，支持国际单位 (1k, 1M, 1G)
  <T>代表时间参数，支持时间单位 (2s, 2m, 2h)
```

2. 看下版本

```
wrk -v

输出：
wrk 4.0.2 [epoll] Copyright (C) 2012 Will Glozer
```
看到是4.0.2版本的wrk，使用了epoll。这意味着我们可以用少量的线程来跟被测服务创建大量连接，进行压测。

3. 做一次简单压测，分析下结果

```
wrk -t8 -c200 -d30s --latency  "http://www.bing.com"

输出：
Running 30s test @ http://www.bing.com
  8 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    46.67ms  215.38ms   1.67s    95.59%
    Req/Sec     7.91k     1.15k   10.26k    70.77%
  Latency Distribution
     50%    2.93ms
     75%    3.78ms
     90%    4.73ms
     99%    1.35s 
  1790465 requests in 30.01s, 684.08MB read
Requests/sec:  59658.29
Transfer/sec:     22.79MB
```

以上使用8个线程200个连接，对bing首页进行了30秒的压测，并要求在压测结果中输出响应延迟信息。以下对压测结果进行简单注释：

```
Running 30s test @ http://www.bing.com （压测时间30s）
  8 threads and 200 connections （共8个测试线程，200个连接）
  Thread Stats   Avg      Stdev     Max   +/- Stdev
              （平均值） （标准差）（最大值）（正负一个标准差所占比例）
    Latency    46.67ms  215.38ms   1.67s    95.59%
    （延迟）
    Req/Sec     7.91k     1.15k   10.26k    70.77%
    （处理中的请求数）
  Latency Distribution （延迟分布）
     50%    2.93ms
     75%    3.78ms
     90%    4.73ms
     99%    1.35s （99分位的延迟）
  1790465 requests in 30.01s, 684.08MB read （30.01秒内共处理完成了1790465个请求，读取了684.08MB数据）
Requests/sec:  59658.29 （平均每秒处理完成59658.29个请求）
Transfer/sec:     22.79MB （平均每秒读取数据22.79MB）
```

可以看到，wrk使用方便，结果清晰。并且因为非阻塞IO的使用，可以在普通的测试机上创建出大量的连接，从而达到较好的压测效果。

### 使用Lua脚本个性化wrk压测

以上两节安装并简单使用了wrk，但这种简单的压测可能不能满足我们的需求。比如我们可能需要使用POST METHOD跟服务器交互；可能需要为每一次请求使用不同的参数，以更好的模拟服务的实际使用场景等。wrk支持用户使用--script指定Lua脚本，来定制压测过程，满足个性化需求。

1. 介绍wrk对Lua脚本的支持

wrk支持在三个阶段对压测进行个性化，分别是启动阶段、运行阶段和结束阶段。每个测试线程，都拥有独立的Lua运行环境。

##### 启动阶段

```
function setup(thread)
```
在脚本文件中实现setup方法，wrk就会在测试线程已经初始化但还没有启动的时候调用该方法。wrk会为每一个测试线程调用一次setup方法，并传入代表测试线程的对象thread作为参数。setup方法中可操作该thread对象，获取信息、存储信息、甚至关闭该线程。

```
thread.addr             - get or set the thread's server address
thread:get(name)        - get the value of a global in the thread's env
thread:set(name, value) - set the value of a global in the thread's env
thread:stop()           - stop the thread
```

##### 运行阶段

```
function init(args)
function delay()
function request()
function response(status, headers, body)
```
init由测试线程调用，只会在进入运行阶段时，调用一次。支持从启动wrk的命令中，获取命令行参数；
delay在每次发送request之前调用，如果需要delay，那么delay相应时间；
request用来生成请求；每一次请求都会调用该方法，所以注意不要在该方法中做耗时的操作；
reponse在每次收到一个响应时调用；为提升性能，如果没有定义该方法，那么wrk不会解析headers和body；

##### 结束阶段

```
function done(summary, latency, requests)
```
该方法在整个测试过程中只会调用一次，可从参数给定的对象中，获取压测结果，生成定制化的测试报告。

##### 自定义脚本中可访问的变量和方法

变量：wrk

```
 wrk = {
    scheme  = "http",
    host    = "localhost",
    port    = nil,
    method  = "GET",
    path    = "/",
    headers = {},
    body    = nil,
    thread  = <userdata>,
  }
```
一个table类型的变量wrk，是全局变量，修改该table，会影响所有请求。

方法：wrk.fomat wrk.lookup wrk.connect

```
  function wrk.format(method, path, headers, body)

    wrk.format returns a HTTP request string containing the passed parameters
    merged with values from the wrk table.
    根据参数和全局变量wrk，生成一个HTTP rquest string。

  function wrk.lookup(host, service)

    wrk.lookup returns a table containing all known addresses for the host
    and service pair. This corresponds to the POSIX getaddrinfo() function.
    给定host和service（port/well known service name），返回所有可用的服务器地址信息。

  function wrk.connect(addr)

    wrk.connect returns true if the address can be connected to, otherwise
    it returns false. The address must be one returned from wrk.lookup().
    测试与给定的服务器地址信息是否可以成功创建连接
```

2. 示例

##### 使用POST METHOD

```
wrk.method = "POST"
wrk.body   = "foo=bar&baz=quux"
wrk.headers["Content-Type"] = "application/x-www-form-urlencoded"
```
通过修改全局变量wrk，使得所有请求都使用POST方法，并指定了body和Content-Type头。


##### 为每次request更换一个参数

```
request = function()
   uid = math.random(1, 10000000)
   path = "/test?uid=" .. uid
   return wrk.format(nil, path)
end
```
通过在request方法中随机生成1~10000000之间的uid，使得请求中的uid参数随机。


##### 每次请求之前延迟10ms

```
function delay()
   return 10
end
```

##### 每个线程要先进行认证，认证之后获取token以进行压测

```
token = nil
path  = "/authenticate"

request = function()
   return wrk.format("GET", path)
end

response = function(status, headers, body)
   if not token and status == 200 then
      token = headers["X-Token"]
      path  = "/resource"
      wrk.headers["X-Token"] = token
   end
end
```
在没有token的情况下，先访问/authenticate认证。认证成功后，读取token并替换path为/resource。

##### 压测支持HTTP pipeline的服务

```
init = function(args)
   local r = {}
   r[1] = wrk.format(nil, "/?foo")
   r[2] = wrk.format(nil, "/?bar")
   r[3] = wrk.format(nil, "/?baz")

   req = table.concat(r)
end

request = function()
   return req
end
```
通过在init方法中将三个HTTP request请求拼接在一起，实现每次发送三个请求，以使用HTTP pipeline。

### 最后

源码非常简洁，简单读了读，很佩服wrk的作者。