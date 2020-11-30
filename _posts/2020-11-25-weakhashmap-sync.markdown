---
layout: post
title:  "WeakHashMap为什么synchronized on queue"
date:   2020-11-25 18:24:26 +0800
categories: java
---

WeakHashMap会在几个合适的时机清洗那些已经被垃圾回收的key, 具体函数是:

    private void expungeStaleEntries() {
        for (Object x; (x = queue.poll()) != null; ) {
            synchronized (queue) {
                ... // 具体的移除逻辑
            }
        }
    }


这个函数很奇怪的一个地方是它synchronized on queue这个变量  
weakhashmap本身不支持线程安全, 其它put, remove方法都没有同步, 这个synchronized是整个类中唯一一处同步  
所以为什么同步呢???  
是ReferenceQueue需要同步吗? 实际上稍微看一下RQ的代码会发现, 当x从queue中poll出来之后, queue就不保持对x的引用了, 而且RQ的内部的同步是同步在一个lock对象, 而非自身. RQ的文档没有特别说要这样做

搜了一下发现原因是这个: [https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6425537](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6425537)

虽然weakhashmap不是线程安全的, 但是大部分java的集合类在多线程中读内容是可以保证安全的, 这个其实很正常

而weakhashmap, 它的size(), get()等方法都会先清理一下weakreference, 所以如果从多个线程查询一下size, 就会出现多个线程并发修改数据结构的情况, 所以这里使用queue同步是为了保证读的线程安全. 至于用queue这个对象作为锁, 只是方便