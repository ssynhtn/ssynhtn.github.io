---
layout: post
title:  "Activity.onTouchEvent的调用顺序"
date:   2020-09-07 18:21:26 +0800
categories: android
---

有一种面试题会问题Activity.onTouchEvent, View.onTouchEvent, OnTouchListener.onTouchEvent的调用顺序  
TLDR: OnTouchListener.onTouchEvent > View.onTouchEvent > Activity.onTouchEvent

有一件事情我们可以看成是已知的: View和Activity中的事件是从ViewRootImpl这个类出发调用的  
我需要从这个起点开始, 其它的太底层了

ViewRootImpl对象持有DecorView mView对象  
但是如果在ViewRootImpl的代码里搜索mView.dispatchTouchEvent会发现没有这个调用  
实际运行会发现实际调用的是View.dispatchPointerEvent这个方法

    public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }

接着进入DecorView的dispatch方法:

      public boolean dispatchTouchEvent(MotionEvent ev) {
          final Window.Callback cb = mWindow.getCallback();
          return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                  ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
      }

说明Window的Callback对象拥有优先处理权  
而Activity的attach方法中创建了PhoneWindow对象并且设置了自身为Callback:  

    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);

说明dispatch首先会到activity中  

    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }

但是activity又首先把处理权交给了window的superDispatchTouchEvent方法, 这个方法会调用decor的同名方法, 接着会调用到默认的View的dispatchTouchEvent方法


换句话说, 如果我们不覆盖Activity的dispatch方法, 那么Activity.onTouchEvent的优先级是最低的

View.onTouchEvent中处理顺序是这样的:  

    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
        result = true;
    }

    if (!result && onTouchEvent(event)) {
        result = true;
    }

说明listenr的优先级更高, 这样处理的原因是当View的内部实现是不能更改的时候, 我们可以通过设置listener来覆盖view的默认行为