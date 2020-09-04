---
layout: post
title:  "命令行查看app冷启动时间"
date:   2020-09-04 18:21:26 +0800
categories: adb
---

adb shell am start -W com.haodingdan.sixin/.ui.splash.SplashActivity

Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.haodingdan.sixin/.ui.splash.SplashActivity }  
Status: ok  
Activity: com.haodingdan.sixin/.ui.MainActivity  
ThisTime: 2519  // 最后一个Activity启动消耗的时间  
TotalTime: 4075  // app冷启动消耗的时间  
WaitTime: 4117  // 据说是: ActivityManagerService启动App的Activity时的总时间（包括当前Activity的onPause()和自己Activity的启动）  
Complete  

根据文档, 这个TotalTime包含了这些部分的内容:  
  * 启动进程。
  * 初始化对象。
  * 创建并初始化 Activity。 // 如果第一个activity是没有view的splash的话会等到有view的activity
  * 扩充布局。
  * 首次绘制应用。

ref: https://developer.android.com/topic/performance/vitals/launch-time  

在logcat中, 系统也会打印相关的信息  
system_process I/ActivityManager: Displayed com.haodingdan.sixin/.ui.MainActivity: +2s960ms (total +3s592ms)

其中第一个时间是ThisTime, total是TotalTime(当activity数不是1个的时候会加上, 貌似), 对于Splash -> Main这样的设计来说总是有这个total的