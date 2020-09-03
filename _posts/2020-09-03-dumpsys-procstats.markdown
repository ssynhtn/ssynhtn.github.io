---
layout: post
title:  "命令行查看app内存占用"
date:   2020-09-03 18:21:26 +0800
categories: adb
---

获取过去一小时内，系统内进程的内存占用  
adb shell dumpsys procstats --hours 1

* com.haodingdan.sixin:channel / u0a105 / v65:  
         TOTAL: 87% (39MB-42MB-43MB/29MB-31MB-32MB/117MB-120MB-121MB over 7)  
       Service: 87% (39MB-42MB-43MB/29MB-31MB-32MB/117MB-120MB-121MB over 7)  
    Service Rs: 0.03%  
* com.haodingdan.sixin / u0a105 / v65:  
         TOTAL: 85% (87MB-156MB-235MB/69MB-123MB-203MB/172MB-264MB-344MB over 588)  
           Top: 85% (87MB-156MB-235MB/69MB-123MB-203MB/172MB-264MB-344MB over 588)  
    Service Rs: 0.02%  
    (Last Act): 2.7%  


每个进程会显示3组内存占用, 每组中的三个数字是最小值，平均值，最大值, 三种占用是PSS, USS, RSS, 合起来是：  
（minPSS-avgPSS-maxPSS/minUSS-avgUSS-maxUSS/minRSS-avgRSS-maxRSS over 样本数）

  * USS: Unique Set Size 独占内存大小（不包含共享库占用的内存），如果这个app被杀死，那么这部分内存会被释放
  * PSS: Proportional Set Size 按比例分摊的内存大小（比例分配共享库占用的内存）  
共享库占用的内存的“按比例分配”到底是怎么分配，据说是按进程数均摊：  
Pss is the amount of memory shared with other processes, 
accounted in a way that the amount is divided evenly between the processes that share it.  
只要有一个进程使用共享库，它就会一直占用内存。（共享库在没有进程需要它后，它是否会释放？）
  * RSS: Resident Set Size 常驻内存大小（包含共享库占用的内存），这个是不准确的


所以内存占用大小有如下规律：RSS >= PSS >= USS

Android studio的memory profiler的时间线上显示的内存，理论上只能是USS或者RSS, 因为你不能显示半个object的占用啊，对比上面的数据，应该是USS, 确实没有必要显示共享库的对象
