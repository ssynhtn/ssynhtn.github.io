---
layout: post
title:  "alias"
date:   2020-09-12 18:21:26 +0800
categories: bash
---

ls --color=auto

没有任何效果

印象中这个--color==auto的设置是因为ls的一个alias而自带了

获取当前的所有alias:

    alias

输出

    alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'
    alias egrep='egrep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias grep='grep --color=auto'
    alias l='ls -CF'
    alias la='ls -A'
    alias ll='ls -alF'
    alias ls='ls --color=auto'
    alias open='xdg-open'

可以看到ls确实是自动加了color

顺便, 将输入复制的剪切板的命令(很长, 这还有用吗)

    your_command | xclip -selection clipboard

据说这些alias是.bashrc中设置的, 那么

    cat ~/.bashrc | grep alias

输出

    # enable color support of ls and also add handy aliases
    alias ls='ls --color=auto'
    #alias dir='dir --color=auto'
    #alias vdir='vdir --color=auto'
    alias grep='grep --color=auto'
    alias fgrep='fgrep --color=auto'
    alias egrep='egrep --color=auto'
    # some more ls aliases
    alias ll='ls -alF'
    alias la='ls -A'
    alias l='ls -CF'
    # Add an "alert" alias for long running commands.  Use like so:
    alias alert='notify-send --urgency=low -i "$([ $? = 0 ] && echo terminal || echo error)" "$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert$//'\'')"'
    # ~/.bash_aliases, instead of adding them here directly.
    if [ -f ~/.bash_aliases ]; then
        . ~/.bash_aliases
    alias open=xdg-open

.bashrc在我的机器上似乎是从.profile导入的

.profile内容:

    # if running bash
    if [ -n "$BASH_VERSION" ]; then
        # include .bashrc if it exists
        if [ -f "$HOME/.bashrc" ]; then
        . "$HOME/.bashrc"
        fi
    fi
