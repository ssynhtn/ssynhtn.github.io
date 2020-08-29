---
layout: post
title:  "处理hex的一个陷阱"
date:   2020-08-26 18:20:26 +0800
categories: java
---

类似于8c4e2629这样的8位的十六进制数据，要转成一个int, 用Integer.parseInt()就可以了：

    int value = Integer.parseInt("8c4e2629");

结果竟然报错了
对parseInt的代码仔细看了半天，发现是因为这个数超出int的表示范围了：

    if (result < multmin) {
        throw NumberFormatException.forInputString(s);
    }

对于8c4e2629这个hex字符串，其实我想要的是1000开头的一个bit string，它的int值是一个负数
但是parseInt是按照人阅读数字的规则去解析的，它判断字符串代表的数为负数的唯一标准是第一个字母为负号
因此8c4e2629这个字符串在它的处理逻辑中是按照正数去处理的，因此就超出范围了

一个解决办法是，对于我的使用场景来说，用Long来解析就可以

    int value = (int) Long.parseLong("8c4e2629");

原因是long到int的强转是直接去除多余的bits的

也许直接操作bits更好：

    public static int hexToInt(String hex) {
        int value = 0;
        for (int i = 0; i < hex.length(); i++) {
            value = (value << 4) ^ Character.digit(hex.charAt(i), 16);
        }

        return value;
    }

