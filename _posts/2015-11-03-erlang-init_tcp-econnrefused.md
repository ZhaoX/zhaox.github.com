---
layout: post
title: "解决erlang节点启动失败报["inet_tcp",econnrefused]的问题"
description: ""
category: Erlang
tags: [Erlang, EPMD]
---
{% include JB/setup %}

今天有同事说他机器上的leofs启动不了。我用console起了一下，发现报如下错：

``` erlang
{error_logger,{{2015,11,3},{6,23,6}},"Protocol: ~tp: register/listen error: ~tp~n",["inet_tcp",econnrefused]}
{error_logger,{{2015,11,3},{6,23,6}},crash_report,[[{initial_call,{net_kernel,init,['Argument__1']}},{pid,<0.601.0>},{registered_name,[]},{error_info,{exit,{error,badarg},[{gen_server,init_it,6,[{file,"gen_server.erl"},{line,322}]},{proc_lib,init_p_do_apply,3,[{file,"proc_lib.erl"},{line,237}]}]}},{ancestors,[net_sup,kernel_sup,<0.591.0>]},{messages,[]},{links,[#Port<0.1236>,<0.598.0>]},{dictionary,[{longnames,true}]},{trap_exit,true},{status,running},{heap_size,610},{stack_size,27},{reductions,774}],[]]}
{error_logger,{{2015,11,3},{6,23,6}},supervisor_report,[{supervisor,{local,net_sup}},{errorContext,start_error},{reason,{'EXIT',nodistribution}},{offender,[{pid,undefined},{name,net_kernel},{mfargs,{net_kernel,start_link,[['manager_0@192.168.1.132',longnames]]}},{restart_type,permanent},{shutdown,2000},{child_type,worker}]}]}
{error_logger,{{2015,11,3},{6,23,6}},supervisor_report,[{supervisor,{local,kernel_sup}},{errorContext,start_error},{reason,{shutdown,{failed_to_start_child,net_kernel,{'EXIT',nodistribution}}}},{offender,[{pid,undefined},{name,net_sup},{mfargs,{erl_distribution,start_link,[]}},{restart_type,permanent},{shutdown,infinity},{child_type,supervisor}]}]}
...
```
怀疑是EPMD有问题。尝试erl，可以正常启动；尝试erl -name test@127.0.0.1，报类似的错误。

尝试epmd -debug，得到如下信息：

```
epmd: Tue Nov  3 09:00:04 2015: epmd running - daemon = 0
epmd: Tue Nov  3 09:00:04 2015: there is already a epmd running at port 4369
```

尝试ps axu | grep epmd，没有发现进程：

```
root     16308  0.0  0.0 112640   976 pts/6    S+   09:00   0:00 grep --color=auto epmd
```

尝试lsof -i:4369，发现如下进程占用了4369端口：

```
COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
docker  25840 root    4u  IPv4  80940      0t0  TCP registry.xxxxxx.com:epmd (LISTEN)
```

杀掉该进程后，恢复正常。确定了问题是EPMD的默认端口号被其它进程占用导致。解决方案：可以在启动EPMD时指定其端口号，也可以设置环境变量ERL_EPMD_PORT来指定其端口号。但是这样治标不治本，合理的做法还是规划好服务器上要启动的服务及其需要使用的端口号，避免冲突。

关于EPMD的介绍和使用，可以看这里：
http://www.erlang.org/doc/man/epmd.html
