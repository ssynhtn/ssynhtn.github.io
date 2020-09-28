---
layout: post
title:  "Choreographer, Message.setAsynchronous, 60fps的谎言, onCreate中post一个Runnable能否获取View的高度"
date:   2020-09-28 18:24:26 +0800
categories: android
---

在Android开发中, 我总是会碰到很多很多的疑问, 今天我对其中几个问题有了初步的理解, 它们是:
  * Message中的setAsynchronous方法是干什么用的
  * Google似乎曾经说过Android能达到60fps, 为什么我的实际体验不是这样的
  * 在logcat中经常提醒掉帧的Choreographer到底是干什么的 
  * 最后, 一个小问题, 在onCreate中post一个Runnable, 在run方法中调用View.getHeight能否保证获取View的高度, 