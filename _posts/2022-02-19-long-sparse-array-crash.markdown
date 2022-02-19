---
layout: post
title:  "ArrayIndexOutOfBoundsException in LongSparseArray"
date:   2022-02-19 18:24:26 +0800
categories: concurrency
---

想记录一下复现一个crash的过程, 又不想写了, 随便写一下.

人在家中坐, 锅从天上来

一个ticket被assign到头上, 是一个crash, 内容类似这样:

```
java.lang.ArrayIndexOutOfBoundsException: src.length=23 srcPos=0 dst.length=51 dstPos=0 length=24
        at java.lang.System.arraycopy(Native Method)
        at com.android.internal.util.GrowingArrayUtils.insert(GrowingArrayUtils.java:142)
        at android.util.LongSparseArray.put(LongSparseArray.java:238)
        at 其它代码
```

LongSparseArray怎么会crash呢, 稍微看一下LongSparseArray的代码, 发现没有任何支持多线程的迹象, 而且这种arraycopy出现length > src.length的情况, 有可能是多线程导致数据corrupt掉了. 所以基本上猜测是调用的代码有问题.

解决办法是在LongSparseArray的访问处加log, 记录非主线程的调用, 这样就能找到错误的调用代码, 锅就可以挪开了, 开心

但是这个ArrayIndexOutOfBoundsException怎样会出现呢, 假如说我们可以以某种方式复现这个crash, 那我们上面这个猜测就有了一定证据, 否则的话, 即使我们找到了非主线程调用的代码, 
也没有证据说它会导致crash的, 毕竟这个crash总共也就出现一次, 手动在生产app上复现几乎没有可能

那就在demo app里尝试手动复现一下吧

首先需要阅读一下LongSparseArray全文...

它使用了二分查找来实现了一个比较简单的map功能

相比于HashMap和TreeMap, 优点是比较节约内存, 因为HashMap有很多空slot, 并且这两个都有Node结构会占据额外的内存

而sparse array就用了两个数组保存keys和values, 同时在插入数据时保持keys的递增, 这样在读操作就可以用二分查找来达到O(logN)的性能, 缺点是写操作的平均时间是O(N), 这个和ArrayList是一样的

具体看一下LongSparseArray.put以及GrowingArrayUtils.insert的代码, 根据stacktrace指示的行数标出了crash的代码

```
// 简化版
public void put(long key, E value) {
    int i = ~ContainerHelpers.binarySearch(mKeys, mSize, key);

    mKeys = GrowingArrayUtils.insert(mKeys, mSize, i, key);
    mValues = GrowingArrayUtils.insert(mValues, mSize, i, value);   // A: crash here
    mSize++;
}
```

```
public static <T> T[] insert(T[] array, int currentSize, int index, T element) {
    if (currentSize + 1 <= array.length) {
        ...
        return array;
    }

    T[] newArray = ArrayUtils.newUnpaddedArray((Class<T>)array.getClass().getComponentType(),
            growSize(currentSize));
    System.arraycopy(array, 0, newArray, 0, index); // B: crash here
    ...
}
```

在A行的i变成B行的index = 24, 而mValues变成array是一个长度为23的数组

i是通过二分查找得到的插入index: ~ContainerHelpers.binarySearch(mKeys, mSize, key); 而这行代码, 如果去研究一下会发现i <= mSize是永远成立的

这个不难理解, 因为在[0, mSize)这个范围二分查找, 要么找到目标值, 那么下标应该是[0, mSize)这个范围的, 要么找不到需要insert, 那么下标是[0, mSize]这个范围的

换句话说mSize已经至少是24了, 而mValues.length = 23

一开始想到, 是因为第一个线程执行了mSize++, 而且第二个线程还在用老的mValues吧?

试了一下发现不行, 仔细看确实如此, 如果第一个线程执行了mSize++, 那么在第一个线程mValues = GrowingArrayUtils.insert()也已经执行了, 那么mValues也已经应该是至少有长度24的

后来想了一下发现可以这样: 第一个线程执行A的时候先卡住, 切换到第二个线程, 第二个线程执行了若干次put操作, 讲mSize增加和Values扩容, 后面再执行A的时候, 切换到第一个线程返回, 讲原来那个更小的mValues返回, 这个时候再切换回第二个线程, 就会出现i > mValues.length的情况出现了

视频链接: https://github.com/ssynhtn/sparse-array-crash/blob/master/sparse-array-crash.mov