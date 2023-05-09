---
title: "escript之诡秘"
slug: "may-be-a-escript-bug"
date: 2014-10-02
tags: [erlang]
---

escript方便用户将erlang当作script来用([[1]](#[1]), [[2]](#[2]))。一个最简单的
escript可以是这样的：

```erlang
#! /usr/bin/env escript
main(_Args) ->
    io:format("hello world~n").
```

但是如果你是像下面这样写的话，事情就没有那么顺利了:

```erlang
#! /usr/bin/env escript
main(_Args) -> io:format("hello world~n").
```

会得到 `Premature end of file reached` 的错误。


这与escript对文件前三行有特别的处理有关:

+ 第一行通常是shebang，以`%!`开头
+ 第二行可以是emacs的一些设置，以`%%`开头
+ 第三行可以是传递给emulator的选项，以`%%!`开头


下面通过escript源代码，看看具体是怎么处理的。

```erlang
%% Skip header and make a heuristic guess about the script type

parse_header(File, KeepFirst) ->
    LineNo = 1,
    ...
    Line1 = get_line(Fd),
    case classify_line(Line1) of
        shebang ->
            find_first_body_line(Fd, HeaderSz0, LineNo, KeepFirst,
                    #sections{shebang = Line1});
        archive ->
            {HeaderSz0, LineNo, Fd,
                #sections{type = archive}};
        beam ->
            {HeaderSz0, LineNo, Fd,
                #sections{type = beam}};
        _ ->
            find_first_body_line(Fd, HeaderSz0, LineNo, KeepFirst,
                    #sections{})
    end.


classify_line(Line) ->
    case Line of
        "#!" ++ _ -> shebang;
        "PK" ++ _ -> archive;
        "FOR1" ++ _ -> beam;
        "%%!" ++ _ -> emu_args;
        "%" ++ _ -> comment;
        _ -> undefined
    end.
```

`parse_header/2`对`Line1`进行分析，当识别出是archive或者是beam时，可进行下一步处理；否则需要继续寻找头部结束的位置。

```erlang
find_first_body_line(Fd, HeaderSz0, LineNo, KeepFirst, Sections) ->
    {ok, HeaderSz1} = file:position(Fd, cur),
    %% Look for special comment on second line

    Line2 = get_line(Fd),
    {ok, HeaderSz2} = file:position(Fd, cur),
    case classify_line(Line2) of
        emu_args ->
            %% Skip special comment on second line
            
            Line3 = get_line(Fd),
            {HeaderSz2, LineNo + 2, Fd,
                Sections#sections{type = guess_type(Line3),
                comment = undefined,
                emu_args = Line2}};
        Line2Type ->
            %% Look for special comment on third line

            Line3 = get_line(Fd),
            {ok, HeaderSz3} = file:position(Fd, cur),
            Line3Type = classify_line(Line3),
            if
            Line3Type =:= emu_args ->
                %% Skip special comment on third line

                Line4 = get_line(Fd),
                {HeaderSz3, LineNo + 3, Fd,
                    Sections#sections{type = guess_type(Line4),
                    comment = Line2,
                    emu_args = Line3}};
            Sections#sections.shebang =:= undefined,
            KeepFirst =:= true ->
                %% No shebang. Use the entire file

                {HeaderSz0, LineNo, Fd,
                    Sections#sections{type = guess_type(Line2)}};
            Sections#sections.shebang =:= undefined ->
                %% No shebang. Skip the first line

                {HeaderSz1, LineNo, Fd,
                    Sections#sections{type = guess_type(Line2)}};
            Line2Type =:= comment ->
                %% Skip shebang on first line and comment on second

                {HeaderSz2, LineNo + 2, Fd,
                    Sections#sections{type = guess_type(Line3),
                    comment = Line2}};
            true ->
                %% Just skip shebang on first line

                {HeaderSz1, LineNo + 1, Fd,
                    Sections#sections{type = guess_type(Line2)}}
            end
    end.
```

问题就出在`find_first_body_line/5`直接读取Line3，当文件只有两行时会被当成文件
异常结束。


或许可以写个PullRequest？ :)


Ref
===
<span id="[1]">[1]</span> [escript](http://www.erlang.org/doc/man/escript.html)

<span id="[2]">[2]</span> [escript的高级特性](http://blog.yufeng.info/archives/153)

-EOF-
