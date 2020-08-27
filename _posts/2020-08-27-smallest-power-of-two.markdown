---
layout: post
title:  "一个有意思的小算法"
date:   2020-08-27 18:21:26 +0800
categories: algorithm
---

今天看ArrayDeque的实现细节，发现一个有意思的辅助函数，用来计算比n大的最小的2的幂次，大致逻辑是这样的

    private static int calculateSize(int n) {   // n >= 0
        n |= (n >>>  1);
        n |= (n >>>  2);
        n |= (n >>>  4);
        n |= (n >>>  8);
        n |= (n >>> 16);
        n++;

        if (n < 0)   // Too many elements, must back off
            n >>>= 1;// Good luck allocating 2 ^ 30 elements

        return n;
    }