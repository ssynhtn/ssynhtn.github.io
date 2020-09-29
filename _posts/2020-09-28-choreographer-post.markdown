---
layout: post
title:  "Choreographer, Message.setAsynchronous, 60fps的谎言, onCreate中post一个Runnable能否获取View的高度, 以及中国特色社会主义的优越性"
date:   2020-09-28 18:24:26 +0800
categories: android
---

在Android开发中, 我总是会碰到很多很多的疑问, 今天我对其中几个问题有了初步的理解, 它们是:
  * Message中的setAsynchronous方法是干什么用的
  * Google似乎曾经说过Android能达到60fps, 为什么实际体验不是这样的
  * 在logcat中经常提醒掉帧的Choreographer到底是干什么的 
  * 最后, 一个无用问题, 在onCreate中post一个Runnable, 在run方法中调用View.getHeight能否保证获取View的高度

但是总结起来涉及的东西好多, 而且很多都是半懂半不懂的, 先记录一下



Message.setAsynchronous这个方法的名称对我来说有点欺骗性(MessageQueue中还有一个相关的方法postSyncBarrier), 因为异步, 屏障等东西都是多线程才有的东西, 而一个MessageQueue虽然会从多个线程添加message, 但是message的执行是在一个loop中的, 也就是一个线程中的, 那为什么还会有异步呢?

首先MessageQueue中没有使用任何类似于CyclicBarrier之类的并发类, 所谓的syncBarrier只是一个target为null的Message, 它其实是一个”插队信号“  
当MessageQueue.next()方法查看头部的message的时候, 默认情况如果这个message的处理时间戳when已经到了, message会返回它, Looper获取到message后会把它交给target也就是Handler来处理

但是如果头部是一个target为null的Message, 这个时候mq会查找队列中第一个标记为async的消息, 如果它的时间到了, 就返回它  
这个所谓的sync barrier会一直生效, 直到removeSyncBarrier(int token)被调用将它移除

所以这里既没有异步, 也没有同步, 只有优先插队, 很像是中国特色的医院对不对? (假如你对中国的医院有一点点🤏的了解的话)

postSyncBarrier的使用场所是ViewRootImpl的scheduleTraversal方法

    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            ...
        }
    }
  
这个方法在Android 4.0版本的时候还是这样的

    public void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            ...
            sendEmptyMessage(DO_TRAVERSAL);
        }
    }

4.0版的ViewRootImpl当时还是继承了Handler类的(😓), 所以可以直接调用sendEmptyMessage  
新版的把mTraversalRunnable交给了Choreographer处理.  
DO_TRAVERSAL和TraversalRunnable的内容都是执行doTraversal方法

Choreographer.postCallback是不是也把Runnable放到了mq中呢? 没有, 它把Runnable包装了一下放到了自己的一个链表中

    // 这里action就是Runnable
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        synchronized (mLock) {
            ...
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                ...
            }
        }
    }

那这个Runnable什么时候会进入mq, 获得执行的机会呢?
Ans: 当 Choreographer 的内部类 FrameDisplayEventReceiver 收到onVsync消息的时候:

    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ...
        Message msg = Message.obtain(mHandler, this);
        msg.setAsynchronous(true);
        mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
    }

    @Override
    public void run() {
        mHavePendingVsync = false;
        doFrame(mTimestampNanos, mFrame);
    }

onVsync会向mq发送一个async的消息, 包裹了自身(这个receiver顺便实现了Runnable), 它的run方法会调用doFrame, doFrame中会去执行之前保存的TraversalRunnable

Q:
  * 新版的这样做会保证traversal会更快地执行吗?   
    不一定啊, 老的方法直接把runnable发送到了mq中, 而新方法需要等待onVsync
  * 老版本方法反而更快吗?  
    也不是啊, 新版的send的message是async的, 所以一旦onVsync被调用, 它就会插队, 因此优先于于mq中所有其它由你的垃圾代码和你引用的垃圾第三方库send的消息. (貌似底层vsync的频率是16ms一次, onVsync的调用频率似乎不是直接一一对应的, 而是被choreographer调用的scheduleVsync控制的?)
  * 老版本处理方式的问题在哪里?  
    View的traversal没有优先处理权, 一个requestLayout请求可能需要等很多其它的message处理完成才能轮到它
  * 新版处理方式能保证UI的60fps渲染吗?
    并不能, 只要主线程中当前正在处理的message耗时超过16ms, async消息也无能为力, 所以当时project butter号称的60fps, 实际上是一个上限😓


再来看下一个问题, Choreographer什么时候会在logcat中提醒掉帧  
Choreographer中的FrameDisplayEventReceiver在onVsync的时候会记录当时的时间:

    public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
        ...
        mTimestampNanos = timestampNanos;
        mFrame = frame;
        ...
    }

等到到运行run的时候, 会向doFrame方法传递这个时间戳:

    void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        ...
        startNanos = System.nanoTime();
        final long jitterNanos = startNanos - frameTimeNanos;
        if (jitterNanos >= mFrameIntervalNanos) {
            final long skippedFrames = jitterNanos / mFrameIntervalNanos;
            if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                        + "The application may be doing too much work on its main thread.");
            }
            final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
            ...
            frameTimeNanos = startNanos - lastFrameOffset;
        }
        ...
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
        ...
    }

可以看到所谓jitterNanos就是现在时间减去上一个onVsync是记录的时间, 如果主线程中在执行上一个message耗时过长, 那么这个jitterNanos就会超过16ms  
而skippedFrames就是超时的时间除以16ms

最后一个问题, 如果在onCreate中post一个Runnable, 那么在这个Runnable中能否获取View的高度?  
首先获取View的高度的最佳方法应该是ViewTreeObserver的回调, but, 既然问题这么问了, 我们来预测一下结果吧

预测: View的高度在layout后才能获取, layout是ViewRootImpl的traversal的一部分, ViewRootImpl的requestLayout方法会调用scheduleTraversal方法
而这个requestLayout方法的第一次调用路径是:

    ActivityThread.handleResumeActivity
    WindowManagerImpl.addView
    WindowManagerGlobal.addView
    ViewRootImpl.setView
    ViewRootImpl.requestLayout

换句话说, 第一次ViewRootImpl.postSyncBarrier发生在我们自己在onCreate中post的Runnable之后, mq的内容会如下

    onCreate中post的runnable
    syncBarrier
    traversal

那么traversal即使能让async的msg插队, 也是插在sync barrier的位置, 还是晚于Runnable, 因此Runnable运行时layout尚未执行, 因此无法获取View的高度

实际运行了一下, 发现确实是这样的  
再运行一下...发现竟然不是的!?...😺  
多运行几次竟然是有时候能获取, 有时候不能获取???

通过多次运行, 结合getMainLooper().dump(), 以及在打印ViewRootImpl的mTraversalBarrier的数值, 对比后发现:  
如果Activity A启动Activity B的时候, B的onCreate执行的时候, A post的一个syncBarrier还没执行掉, 这样在B的scheduleTraversal&&onVsync调用后, mq的内容会包括这些:

    A syncBarrier
    我们B.onCreate中post的runnable
    B syncBarrier
    B traversal

当looper发现在A的syncBarrier为mq的第一个msg的时候, B的第一个traversal因为是async的, 会优先于我们在B的onCreate中post的runnable执行

当然如果我们在post的runnable中再post一个runnable, 那它是晚于当前activity的scheduleTraversal中post的sync barrier, 因此是晚于layout的执行的






