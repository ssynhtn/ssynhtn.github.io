---
layout: post
title:  "nix快速切换Java Home"
date:   2020-09-22 18:24:26 +0800
categories: bash
---

/usr/libexec/java_home 这个文件可以打印这台设备上安装的jdk, 不知道是为啥, 反正它可以

默认似乎会返回最新的版本?

    /usr/libexec/java_home
    /Library/Java/JavaVirtualMachines/jdk-15.jdk/Contents/Home

-V参数打印所有

    /usr/libexec/java_home -V
    Matching Java Virtual Machines (2):
        15, x86_64:	"Java SE 15"	/Library/Java/JavaVirtualMachines/jdk-15.jdk/Contents/Home
        11.0.8, x86_64:	"Java SE 11.0.8"	/Library/Java/JavaVirtualMachines/jdk-11.0.8.jdk/Contents/Home

    /Library/Java/JavaVirtualMachines/jdk-15.jdk/Contents/Home


快速切换JAVA_HOME
在~/.bash_profile中添加

    alias useJava15="export JAVA_HOME=`/usr/libexec/java_home -v 15`"
    alias useJava11="export JAVA_HOME=`/usr/libexec/java_home -v 11`"
    alias useJava8="export JAVA_HOME=`/usr/libexec/java_home -v 1.8`"
    useJava11


打印jvm使用道垃圾回收算法

    java -XX:+PrintCommandLineFlags -version
    -XX:G1ConcRefinementThreads=10 -XX:GCDrainStackTargetSize=64 -XX:InitialHeapSize=268435456 -XX:MaxHeapSize=4294967296 -XX:+PrintCommandLineFlags -XX:ReservedCodeCacheSize=251658240 -XX:+SegmentedCodeCache -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseG1GC 
    java version "11.0.8" 2020-07-14 LTS
    Java(TM) SE Runtime Environment 18.9 (build 11.0.8+10-LTS)
    Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.8+10-LTS, mixed mode)

结果中+UseG1GC说明java11使用的是G1

测试了一下java15使用的也是G1
jdk8使用的是Parallel GC 
