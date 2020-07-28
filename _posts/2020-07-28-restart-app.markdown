---
layout: post
title: "Restart an app programmatically"
date: 2020-07-28 22:10:25 +0800
categories: android
---

虽然已经有Jake写了一个[ProcessPhoenix](https://github.com/JakeWharton/ProcessPhoenix/blob/master/process-phoenix/src/main/java/com/jakewharton/processphoenix/ProcessPhoenix.java)这样的类来强制重启app了


但是我不是太明白他为什么要在System.exit之后先启动ProcessPhoenix这个activity，并且还是在单独的:phoenix进程中，然后再次System.exit并打开app的默认activity

在我的测试中

    startActivity(Intent.makeRestartActivityTask(  
            getPackageManager().getLaunchIntentForPackage(getPackageName()).getComponent()));   
    System.exit(0);

貌似就足够了