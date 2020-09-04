---
layout: post
title:  "好订单app内存泄漏修复记录"
date:   2020-09-03 18:22:26 +0800
categories: android
---

再次接手好订单app的维护工作后，我给它加了bugsnag的崩溃统计  
结果我发现隔一段时间就报告一个OutOfMemoryError  
我是有点惊讶的, 因为这个app是很轻量级的  
看看是什么问题吧

1, HintViewFragment -> closeView
2, MainActivity -> static PageChanged
3, Toast -> static mCurrentToast
4, static sheng, shi, TextView

to be continued