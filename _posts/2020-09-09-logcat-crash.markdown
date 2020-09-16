---
layout: post
title:  "adb查看崩溃buffer"
date:   2020-09-09 18:21:26 +0800
categories: adb
---

出于各种原因, 有的时候app崩溃的时候, android studio中的logcat里的崩溃日志会找不到

可以使用adb查看崩溃信息

adb log -b crash

根据文档, 系统进程logd会维护若干环形buffer, 默认的三个buffer是main(这个大概app的log都会到这里吧), system(系统日志), crash(崩溃日志)

还有一个有意思的buffer是events, 里面记录的是系统的事件, 其中包括ams启动activity的事件等


https://developer.android.com/studio/command-line/logcat
