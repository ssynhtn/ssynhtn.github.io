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

Android studio的memory profiler的时间线上显示的内存，对比大小可能是USS或者PSS


meminfo可以显示更多详细的数据，甚至还会获取View的个数ViewRootImple(窗口)的个数，打开的数据库的个数
其中“All Dalvik and native heap allocations you make will be private dirty RAM”
Java和native代码在堆中创建的对对象都属于私有脏内存

adb shell dumpsys meminfo com.haodingdan.sixin [-d]

>Applications Memory Usage (in Kilobytes):
>Uptime: 5838138 Realtime: 5838138
>
>** MEMINFO in pid 8552 [com.haodingdan.sixin] **
>                   Pss  Private  Private  SwapPss     Heap     Heap     Heap
>                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
>                ------   ------   ------   ------   ------   ------   ------
>  Native Heap    36767    36484        0        0    50852    48485     2366
>  Dalvik Heap     7300     7004        0        0    13406     7278     6128
> Dalvik Other     2431     2424        0        0                           
>        Stack       84       84        0        0                           
>       Cursor       48       48        0        0                           
>       Ashmem       85        0        0        0                           
>    Other dev       52        0       52        0                           
>     .so mmap    14396      144     5832        0                           
>    .jar mmap     3237        0      508        0                           
>    .apk mmap    21201      124     6456        0                           
>    .ttf mmap      283        0      100        0                           
>    .dex mmap    18480    18112      268        0                           
>    .oat mmap      445        0       64        0                           
>    .art mmap     7049     6224       68        0                           
>   Other mmap     4813      328     1336        1                           
>      Unknown     2573     2560        0        1                           
>        TOTAL   119246    73536    14684        2    64258    55763     8494
> 
> App Summary
>                       Pss(KB)
>                        ------
>           Java Heap:    13296
>         Native Heap:    36484
>                Code:    31608
>               Stack:       84
>            Graphics:        0
>       Private Other:     6748
>              System:    31026
> 
>               TOTAL:   119246       TOTAL SWAP PSS:        2
> 
> Objects
>               Views:      396         ViewRootImpl:        1
>         AppContexts:        5           Activities:        1
>              Assets:        4        AssetManagers:        0
>       Local Binders:       77        Proxy Binders:       39
>       Parcel memory:       22         Parcel count:       92
>    Death Recipients:        2      OpenSSL Sockets:        1
>            WebViews:        1
> 
> SQL
>         MEMORY_USED:     1071
>  PAGECACHE_OVERFLOW:      543          MALLOC_SIZE:      117
> 
> DATABASES
>      pgsz     dbsz   Lookaside(b)          cache  Dbname
>         4       76             98         9/39/7  /data/user/0/com.haodingdan.sixin/databases/androidx.work.workdb
>         4       12                         0/0/0    (attached) temp
>         4      528            109      416/43/23  /data/user/0/com.haodingdan.sixin/databases/user_357262.db
>         4       20             37         1/16/2  /data/user/0/com.haodingdan.sixin/databases/accs.db
