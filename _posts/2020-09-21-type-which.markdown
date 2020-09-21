---
layout: post
title:  "which vs type"
date:   2020-09-21 18:24:26 +0800
categories: bash
---

which是一个很好用的命令, 用它可以知道bash中运行的命令的文件的具体位置, 这对于想知道在混乱的Linux文件系统哪个文件在哪里的人来说太实用了, 虽然说大多数时候知道了文件好像也并没什么卵用, 但是至少会让人产生一种不会迷路的感觉, 而且它的名字也很直观

    which git
    /usr/local/bin/git

今天我知道了一个type命令, 它似乎和which实现类似的功能

    type git
    git is /usr/local/bin/git

但是对于bash的内置命令, which和type返回的结果不一样 

    which cd
    /usr/bin/cd
    type cd
    cd is a shell builtin

所以当运行cd的时候, 这个cd到底是内置命令还是/usr/bin/cd?

实际上cd是一个内置命令

如果直接指明cd的路径来运行cd, 会发现当前的路径根本没有改变, 原因是那那样运行cd会新建一个子进程, 而在子进程中修改路径和当前的进程中的路径没有关系

所以虽然type的名字很难听, 但是似乎type更好呢

附:
which的man page说:

    The which utility takes a list of command names and searches the path for each executable file that would be run had these commands actually been invoked.

所以which就是在path中寻找命令所对应的文件, 至于bash是不是运行了那个命令, 它根本没有保证

type因为本身是bash的一个内置命令, 它的帮助文档在man bash中:

    type [-aftpP] name [name ...]
    With no options, indicate how each name would be interpreted if used as a command name.  If the -t option is used, type prints a string  which  is  one  of alias, keyword, function, builtin, or file

所以type会告诉我们, 当一个命令name在bash中被使用的时候, 它具体代表什么