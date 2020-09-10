---
layout: post
title:  "adb查看崩溃buffer"
date:   2020-09-09 18:21:26 +0800
categories: adb
---

出于各种原因, 有的时候app崩溃的时候, logcat里的崩溃日志会找不到

可以使用adb查看崩溃信息

adb log -b crash

根据文档, 系统进程logd会维护若干环形buffer, 最重要的三个buffer是main(这个大概app的log都会到这里吧), system(系统日志), crash(崩溃日志)


https://developer.android.com/studio/command-line/logcat
