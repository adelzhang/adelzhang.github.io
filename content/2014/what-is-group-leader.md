---
title: group leader二三事 
date: 2014-09-20 20:31
tags: [erlang]
---

## I. group leader在哪

有很多地方可以发现group leader的踪迹，比如在`application`模块中。通常我们调用`application:start(Application)`来启动应用，`application`
是对`application_controller`的封装：

    start(Application, RestartType) ->
        case load(Application) of
            ok ->
                Name = get_appl_name(Application),
                application_controller:start_application(Name, RestartType);
            {error, {already_loaded, Name}} ->
                application_controller:start_application(Name, RestartType);
            Error ->
                Error
        end.

`application_controller`是一个仿造的OTP进程（为什么要仿造，而不是直接使用gen_server或者proc_lib呢？）:

    start(KernelApp) ->
        Init = self(),
        AC = spawn_link(fun() -> init(Init, KernelApp) end),
        ...

    init(Init, KernelApp) ->
        register(?AC, self()),
        process_flag(trap_exit, true),
        put('$ancestors', [Init]), % OTP-5811, for gen_server compatibility
        put('$initial_call', {application_controller, start, 1}),
        ...


`application_controller`确保Application的所有依赖应用都已启动，否则返回`{error, {not_started, App}}`，其中App是没有启动的依赖应用。在
一个新建进程（为什么要在新进程中？），创建属于Application的application master进程:

    handle_call({start_application, AppName, RestartType}, From, S) ->
        ...
        case catch check_start_cond(AppName, RestartType, Started, Running) of
            {ok, Appl} ->
                 spawn_starter(From, Appl, S, normal),
                 {noreply, S#state{starting = [{AppName, RestartType, normal, From} |
                                                Starting],
                                   start_req = [{AppName, From} | Start_req]}};
            {error, _R} = Error ->
                {reply, Error, S}
        end,
        ...

    spawn_starter(From, Appl, S, Type) ->
        spawn_link(?MODULE, init_starter, [From, Appl, S, Type]).

    init_starter(_From, Appl, S, Type) ->
        process_flag(trap_exit, true),
        AppName = Appl#appl.name,
        gen_server:cast(?AC, {application_started, AppName, 
                  catch start_appl(Appl, S, Type)}).

    start_appl(Appl, S, Type) ->
        ...
        application_master:start_link(ApplData, Type),
        ...

group leader马上就要出现了。application master是Application的第一个进程，它会调用Application behaviour回调函数`Module:start/2`:

    start_link(ApplData, Type) ->
        Parent = whereis(application_controller),
        proc_lib:start_link(application_master, init,
                [Parent, self(), ApplData, Type]). 

在`init`函数前的一段注释中，描述了AM(application master)是如何启动应用的:

    %%%-----------------------------------------------------------------
    %%% The logical and physical process structrure is as follows:
    %%%
    %%%         logical                physical
    %%%
    %%%         --------               --------
    %%%         |AM(GL)|               |AM(GL)|               
    %%%         --------               -------- 
    %%%            |                       |
    %%%         --------               --------
    %%%         |Appl P|               |   X  |
    %%%         --------               --------
    %%%                                    |
    %%%                                --------
    %%%                                |Appl P|
    %%%                                --------
    %%%
    %%% Where AM(GL) == Application Master (Group Leader)
    %%%       Appl P == The application specific root process (child to AM)
    %%%       X      == A special 'invisible' process
    %%% The reason for not using the logical structrure is that
    %%% the application start function is synchronous, and 
    %%% that the AM is GL.  This means that if AM executed the start
    %%% function, and this function uses spawn_request/1
    %%% or io, deadlock would occur.  Therefore, this function is
    %%% executed by the process X.  Also, AM needs three loops;
    %%% init_loop (waiting for the start function to return)
    %%% main_loop
    %%% terminate_loop (waiting for the process to die)
    %%% In each of these loops, io and other requests are handled.

    init(Parent, Starter, ApplData, Type) ->
        link(Parent),
        process_flag(trap_exit, true),
        OldGleader = group_leader(),
        group_leader(self(), self()),
        ...

group leader终于出现了，那么疑问就变成：什么是group leader？为什么AM不采用辅助进程的话会造成deadlock？

## II. 什么是group leader

在erlang doc中，有关group leader是这样描述的：

    Every process is a member of some process group and all groups have a group leader. 
    All IO from the group is channeled to the group leader. 
    When a new process is spawned, it gets the same group leader as the spawning process. 
    Initially, at system start-up, init is both its own group leader and the group leader of all processes.

`group_leader()`可以返回当前进程的group leader，`group_leader(GL, Pid)`修改Pid进程的GL。

以io:format/0为例，看看group leader怎么发挥作用。io:format会向GL发送消息，并且等待GL返回。这也解释了AM不采用
辅助进程的话可能造成deadlock的原因。

    format(Format, Args) ->
        format(default_output(), Format, Args).

    default_output() ->
        group_leader().

    format(Io, Format, Args) ->
        o_request(Io, {format,Format,Args}, format).

    o_request(Io, Request, Func) ->
        ...
        request(Io, Request)
        ...

    request(Pid, Request) when is_pid(Pid) ->
        execute_request(Pid, io_request(Pid, Request)),
        ...

    execute_request(Pid, {Convert,Converted}) ->
        Pid ! {io_request,self(),Pid,Converted},
        ...
        wait_io_mon_reply(Pid, Mref),
        ...

    wait_io_mon_reply(From, Mref) ->
        receive
            {io_reply, From, Reply} ->
                erlang:demonitor(Mref, [flush]),
                Reply;
            {'EXIT', From, _What} ->
                receive
                    {'DOWN', Mref, _, _, _} -> true
                after 0 -> true
                end,
                {error,terminated};
            {'DOWN', Mref, _, _, _} ->
                receive
                    {'EXIT', From, _What} -> true
                after 0 -> true
                end,
                {error,terminated}
        end.


## III. 何时需要group leader

一般情况下，我们并不用关心group leader。但在某些场景下，可能需要对io进行拦截，这时候就
需要修改进程的group leader。rpc模块就利用了这种技术，使得在远程节点运行的代码，结果输出
在本地节点。本地node调用rpc时，group_leader()是参数之一。远程node spawn一个新进程来执行
rpc，并将该进程GL修改为本地node提高的group_leader()，从而实现io的拦截。

    %% code snippet of rpc.erl

    call(N,M,F,A) when node() =:= N ->  %% Optimize local call
        local_call(M, F, A);
    call(N,M,F,A) ->
        do_call(N, {call,M,F,A,group_leader()}, infinity).

    do_call(Node, Request, infinity) ->
        rpc_check(catch gen_server:call({?NAME,Node}, Request, infinity));

    handle_call({call, Mod, Fun, Args, Gleader}, To, S) ->
        handle_call_call(Mod, Fun, Args, Gleader, To, S);

    handle_call_call(Mod, Fun, Args, Gleader, To, S) ->
        RpcServer = self(),
        %% Spawn not to block the rpc server.
        {Caller,_} =
        erlang:spawn_monitor(
          fun () ->
              set_group_leader(Gleader),
              Reply = 
                  %% in case some sucker rex'es 
                  %% something that throws
                  case catch apply(Mod, Fun, Args) of
                  {'EXIT', _} = Exit ->
                      {badrpc, Exit};
                  Result ->
                      Result
                  end,
              RpcServer ! {self(), {reply, Reply}}
          end),
        {noreply, gb_trees:insert(Caller, To, S)}.

    set_group_leader(Gleader) when is_pid(Gleader) -> 
        group_leader(Gleader, self());
    set_group_leader(user) -> 
        %% For example, hidden C nodes doesn't want any I/O.
        Gleader = case whereis(user) of
              Pid when is_pid(Pid) -> Pid;
              undefined -> proxy_user()
              end,
        group_leader(Gleader, self()).


在`set_group_leader/1`中，还能看到`user`的身影。官方文档中关于`user`的介绍：

    MODULE SUMMARY
    Standard I/O Server

    DESCRIPTION
    user is a server which responds to all the messages defined in the I/O interface. 
    The code in user.erl can be used as a model for building alternative I/O servers.

所以，user是Erlang/OTP中默认的标准i/o进程，不知道io_request怎么处理时，往user扔应该不会错吧？ :)

-EOF-
