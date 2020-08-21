---
layout: post
title:  "为什么GUI都是单线程的"
date:   2020-08-19 18:20:26 +0800
categories: concurrency
---
如果GUI框架是多线程的

1，回调会造成死锁

2，输入事件从硬件到操作系统到用户代码，如果不是用事件驱动模型，而是在线程中用回调去通知应用，那么会和应用程度代码调用gui sdk的代码所在的线程产生两个方向的调用，可能会造成死锁


ref: 
0, https://community.oracle.com/blogs/kgh/2004/10/19/multithreaded-toolkits-failed-dream
1, Why Threads Are A Bad Idea https://web.stanford.edu/~ouster/cgi-bin/papers/threads.pdf