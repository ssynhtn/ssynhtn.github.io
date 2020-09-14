---
layout: post
title:  "adb获取heap dump文件"
date:   2020-09-14 18:24:26 +0800
categories: leetcode
---

1, 获取heap文件
    
    adb shell am dumpheap com.packagename /data/local/tmp/a.hprof

2, 复制文件到电脑

    adb pull $_ .

$_表示上一个命令的最后一个参数, 省去复制粘贴

3, Android的hprof文件格式似乎和Java的不同, 需要转换

    hprof-conv a.hprof j.hprof

最后获取的j.hprof就可以用mat打开了