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