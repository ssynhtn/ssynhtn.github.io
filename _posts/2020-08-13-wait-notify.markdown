---
layout: post
title:  "Object.wait, Object.notify"
date:   2020-08-13 18:20:25 +0800
categories: android
---
Object.wait()方法

一个thread T要调用一个object的wait方法，首先要获取它的monitor lock:
Causes the current thread to wait until either another thread invokes the notify() method or the notifyAll() method for this object, or a specified amount of time has elapsed. The current thread must own this object's monitor.

调用wait后，T的状态从Thread.State.Runnable转变成Thread.State.Waiting， 并且会释放这个object的monitor:
This method causes the current thread (call it T) to place itself in the wait set for this object and then to relinquish any and all synchronization claims on this object.

等另一个拥有这个object的monitor的线程S(This method should only be called by a thread that is the owner of this object's monitor.)调用object.notify()或者notifyAll(）之后，notify会选择一个waiting状态的thread转换成block状态，

notifyall会转换所有waiting在这个object上的thread，但是此时对这些跳出waiting状态对thread来说，wait方法还未结束，因为

1，S还持有monitor，它调用完notify后它的synchronize块还很可能未结束
2，即使S跳出了同步块，多个转换成block状态，已经其它在运行的想要执行synchronize方法/block的线程，只有一个会获取monitor

等到从waiting转换到block状态到thread获取了monitor，它的wait方法才正式结束，此时它的状态和调用wait方法之前完全一样:

once it has gained control of the object, all its synchronization claims on the object are restored to the status quo ante - that is, to the situation as of the time that the wait method was invoked. Thread T then returns from the invocation of the wait method. Thus, on return from the wait method, the synchronization state of the object and of thread T is exactly as it was when the wait method was invoked.

Q：为什么调用wait方法需要首先拿到对象的锁？
A：常用的wait的使用方式是:

    synchronized(lock) {
	    while (!condition) {
		    try {
			    lock.wait();
		    } catch (ignored) {}
	    }
    }

这种方式中condition变量是被lock保护的，当当前线程结束wait时，持有lock的锁，因此可以安全地去检查condition

  

如果Java不这样要求，那么wait结束，检查condition为true后，有可能此时会切换到别的线程修改了condition

  

Java语言的设计者到底是怎么想的，貌似无从得知

更多的解释和例子：https://stackoverflow.com/questions/2779484/why-must-wait-always-be-in-synchronized-block#:~:text=The%20wait()%20is%20called,inside%20a%20synchronized%20method%2Fblock.