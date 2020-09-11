---
layout: post
title:  "无用知识之获取对象内存占用大小"
date:   2020-09-11 18:21:26 +0800
categories: java
---

无脑记录一下

首先是这个工具类

    public class ObjectSizeFetcher {
        private static Instrumentation instrumentation;

        public static void premain(String args, Instrumentation inst) {
            instrumentation = inst;
        }

        public static long getObjectSize(Object o) {
            return instrumentation.getObjectSize(o);
        }

        public static void main(String[] args) {
            String filename = args.length > 0 ? args[0] : "pinyin.properties";
            Properties pairs = PinyinResource.getResource(filename);
            long size = getObjectSize(pairs);
            long total = 0;
            long entryTotal = 0;
            for (Map.Entry<Object, Object> entry : pairs.entrySet()) {
                Object key = entry.getKey();
                Object value = entry.getValue();
                total += getObjectSize(key);
                total += getObjectSize(value);
                entryTotal += getObjectSize(entry);
            }

            System.out.println("used bytes of props " + size + ", of entries " + entryTotal
                    + ", of object values " + total);

            for (int i = 1; i < args.length; i++) {
                System.out.println(args[i] + " takes " + getObjectSize(args[i]) + " bytes, chars tooks " + getObjectSize(args[i].toCharArray()));
            }

        }
    }

其次是编译成class文件 -> 用idea的运行  
或者

    javac dir/*.java -d dir

创建一个声明了premain和main类的manifest.mf文件

    Manifest-Version: 1.0
    Main-Class: com.ssynhtn.text.ObjectSizeFetcher
    Premain-Class: com.ssynhtn.text.ObjectSizeFetcher

打包

    jar cmf dir/manifest.mf hello.jar dir/*.class

指定agent运行, 至于agent是啥先不管了, 反正是和premain类组合, 会先调用premian方法的一个东西

    java -javaagent:hello.jar -jar hello.jar args

运行结果

    used bytes of props 48, of entries 668896, of object values 1003344
    hello takes 24 bytes, chars tooks 32
    worldworl takes 24 bytes, chars tooks 40
    我 takes 24 bytes, chars tooks 24

2万个key->value组成的Properties类, 如果只计算它的entry和string, 至少占用了1.6MB的内存

似乎好像有点对不上呢, 不过先不管了, 总之String的内存占用比想象的要高的样子