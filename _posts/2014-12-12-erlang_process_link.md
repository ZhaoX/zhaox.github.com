---
layout: post
title: "Erlang进程的Link机制"
description: ""
category: Erlang 
tags: [Erlang]
---
{% include JB/setup %}

**这篇文件还不是最终版，有时间时，我会再来补充完善。**

## 什么是link

Erlang程序基于进程建模，进程之间的交互机制有收发消息，link和monitor。其中，收发消息通常用于正常的进程间通讯，而link和monitor多用于异常情况处理，本文从应用的角度介绍和分析link机制。link是双向全联通的，用来将两个或多个进程绑定在一起，绑定在一起之后，VM会保证在有进程退出时，对与其绑定在一起的进程执行特定的操作。

## 创建link和取消link

> Two processes can be linked to each other. A link between two
> processes Pid1 and Pid2 is created by Pid1 calling the BIF
> link(Pid2)(or vice versa). There also exists a number a spawn_link
> BIFs, which spawns and links to a process in one operation.
> 
> Links are bidirectional and there can only be one link between two
> processes.Repeated calls to link(Pid) have no effect.

> A link can be removed bycalling the BIF unlink(Pid).

## 当有进程退出时

#### 发送Exit Signal

> When a process terminates, it will terminate with an exit reason as
> explained in Process Termination above. This exit reason is emitted in
> an exit signal to all linked processes.
> 
> A process can also call the function exit(Pid,Reason). This will
> result in an exit signal with exit reason Reason being emitted to Pid,
> but does not affect the calling process.

#### Exit Signal的默认处理方式

> The default behaviour when a process receives an exit signal with an
> exit reason other than normal, is to terminate and in turn emit exit
> signals with the same exit reason to its linked processes. An exit
> signal with reason normal is ignored

#### 将Exit Signal转换为普通的进程消息

> A process can be set to trap exit signals by calling:
> 
> process_flag(trap_exit, true)
> 
> When a process is trapping exits, it will not terminate when an exit
> signal is received. Instead, the signal is transformed into a
> message{'EXIT',FromPid,Reason} which is put into the mailbox of the
> process just like a regular message.
> 
> An exception to the above is if the exit reason is kill, that is if
> exit(Pid,kill) has been called. This will unconditionally terminate
> the process, regardless of if it is trapping exit signals or not.

## link与OTP的关系

OTP作为Erlang官方的编程框架被广泛应用，在OTP的实现中，link机制被广泛的应用。

> Erlang has a built-in feature for error handling between processes.
> Terminating processes will emit exit signals to all linked processes,
> which may terminate as well or handle the exit in some way. This
> feature can be used to build hierarchical program structures where
> some processes are supervising other processes, for example restarting
> them if they terminate abnormally.
> 
> Refer to OTP Design Principles for more information about OTP
> supervision trees, which uses this feature.

#### gen_server

假设存在a，b两个进程，其中b是gen_server。我们在进程a中调用b:start_link()，使两个进程link在一起，然后来讨论一些异常情况。

- *a进程正常退出 -> b进程正常运行*
- *a进程异常退出 -> b进程退出*
- *a进程正常退出, b进程中调用了process_flag(trap_exit, true) ->  b进程不会收到exit msg，退出*
- *a进程异常退出, b进程中调用了process_flag(trap_exit, true) ->  b进程不会收到exit msg，退出*
- *b进程正常退出 -> a进程正常运行*
- *b进程异常退出 -> a进程退出*
- *b进程正常退出, a进程中调用了process_flag(trap_exit, true) ->  a进程收到{'EXIT',Pid_b,normal}*
- *b进程异常退出, a进程中调用了process_flag(trap_exit, true) ->  a进程收到{'EXIT',Pid_b,Reason}*

看起来第3项和第4项不太正常，似乎跟刚刚介绍的erlang link机制冲突了。出现这种现象的原因，是gen_server不是普通进程，它在一个普通的进程上，添加一些默认的行为，具体到这个问题，就是gen_server在收到来自父进程（调用start_link>的进程）的{'EXIT',Pid_Parent,Reason}

```erlang
decode_msg(Msg, Parent, Name, State, Mod, Time, Debug, Hib) ->
     case Msg of
         {system, From, get_state} ->
             sys:handle_system_msg(get_state, From, Parent, ?MODULE, Debug,
                                   {State, [Name, State, Mod, Time]}, Hib);
         {system, From, {replace_state, StateFun}} ->
             NState = try StateFun(State) catch _:_ -> State end,
             sys:handle_system_msg(replace_state, From, Parent, ?MODULE, Debug,
                                   {NState, [Name, NState, Mod, Time]}, Hib);
         {system, From, Req} ->
             sys:handle_system_msg(Req, From, Parent, ?MODULE, Debug,
                                   [Name, State, Mod, Time], Hib);
         {'EXIT', Parent, Reason} ->
             terminate(Reason, Name, Msg, Mod, State, Debug);
         _Msg when Debug =:= [] ->
             handle_msg(Msg, Parent, Name, State, Mod);
         _Msg ->
             Debug1 = sys:handle_debug(Debug, fun print_event/3,
                                       Name, {in, Msg}),
             handle_msg(Msg, Parent, Name, State, Mod, Debug1)
     end.
```

## link与业务建模
待续




