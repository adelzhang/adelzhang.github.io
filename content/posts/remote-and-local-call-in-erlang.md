---
title: "函数调用和热更新" 
slug: "remote-and-local-call-in-erlang"
date: 2014-09-06
tags: ["erlang"]
---

erlang中函数调用分为两种：

1. remote call/external function call: `m:f(e1, ..., eN)`
2. local call: `f(e1, ..., eN)`

remote call 和 local call的主要差别在热更新时。erlang运行时对module可以维护
两个版本的代码:`current`和 `old`。老版本和新版本可以同时运行：remote call总是使用
`current`版本，local call可能运行`old`版本。当有更新的版本代码加载时，运行`old`
代码的进程会终止。可以通过调用remote call，使进程切换到 `current` 版本。

	-module(m).
	-export([loop/0]).

	loop() ->
    	receive
   	    	code_switch ->
            	m:loop();
        	Msg ->
				do_something,
            	loop()
    	end.

这种差别也体现在spawn函数调用时，`spawn(M, F, A)` 使用`current`代码，而 `spawn(Fun)` 中 `Fun` 是local call，不能进行热更新。 

但是存在这样一个问题：当进行热更新时，怎么保证运行老版本代码的进程不死掉呢？


## --- Ref ---
[1] [Function Calls](http://erlang.org/doc/reference_manual/expressions.html#id78068)

[2] [Code Loading](http://erlang.org/doc/reference_manual/code_loading.html#id86334)
