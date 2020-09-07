---
layout: post
title:  "Activity.onTouchEvent的调用顺序"
date:   2020-09-07 18:21:26 +0800
categories: android
---

有一种面试题会问题Activity.onTouchEvent, View.onTouchEvent, OnTouchListener.onTouch的调用顺序  
太长不看: OnTouchListener.onTouch > View.onTouchEvent > Activity.onTouchEvent

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

View.onDispatchTouchEvent中处理顺序是这样的:  

    ListenerInfo li = mListenerInfo;
    if (li != null && li.mOnTouchListener != null
            && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnTouchListener.onTouch(this, event)) {
        result = true;
    }

    if (!result && onTouchEvent(event)) {
        result = true;
    }

说明listener的优先级更高, 这样处理的原因是当View的内部实现是不能更改的时候, 我们可以通过设置listener来覆盖view的默认行为


另一个问题是View.OnClickListener是如何被触发的, 观看代码后会发现, 这里涉及的东西其实也不少, 但是最常出现的流程其实只有两步

一个最简单的流程是这样的:  

    View.onTouchEvent:
        ACTION_DOWN: setPressed()
        ACTION_UP: post(mPerformClick) || performClickInternal();

    performClickInternal -> performClick -> onClickListener != null && onClickListener.onClick(this) 

onTouchEvent -> onClick 的流程中考虑了这些东西:  
第一:
  * View是否是clickable的, 是否是enabled
    * 如果!clickable的 => 没有操作, 并且不消耗touch事件, 返回false
    * 如果是clickable的
      * 如果disabled => 没有操作, 但是消耗touch事件, 返回true, 这个有点儿奇怪, 可是它就是这么设定的
      * enabled && clickable => 这种情况才会处理touch事件
    
View的默认clickable是从View_clickable属性读取的, 常见的控件中Button, ImageButton, EditText是默认clickable的  
如果查看过Button的源码会发现这个类继承TextView, 但是却几乎没有任何改变, 最重要的改动是它的第二个构造方法(inflater调用的构造方法)中, 指定了从当前主题的buttonStyle中读取属性
而默认主题的buttonStyle指向了Widget.Button, 其中设置了clickable:

    <style name="Theme">
      <item name="buttonStyle">@style/Widget.Button</item>
    </style>
    <style name="Widget.Button">
        ...
        <item name="focusable">true</item>
        <item name="clickable">true</item>
        ...
    </style>

可是button的默认background太丑了, 所以我一般用textview来代替button, 在设置onClickListener的时候clickable也会被设为true:

    public void setOnClickListener(@Nullable OnClickListener l) {
        if (!isClickable()) {
            setClickable(true);
        }
        getListenerInfo().mOnClickListener = l;
    }


第二:
其次会考虑的是View是否在一个可以滚动的parent中, 如果不是的, 那么会直接setPressed(true, x, y), 如果是的话, 那不会直接调用, 会设置为PrePressed状态, 并postDelay一个CheckForTap对象, 如果100ms后这个对象还没有被移除, 说明此时手势还在进行, 这时CheckForTap.runnable会调用setPressed(true, x, y)

这个机制可以解释为什么在ListView/RecyclerView中快速滚动时, item的background不会触发pressed状态, 但是如果很慢地挪动, 那么手指下方地view的pressed状态会被触发

判断是否在“可以滚动的parent”中的方法是isInScrollingContainer, 它其实是递归调用parent的shouldDelayChildPressedState方法, 这个方法在ViewGroup的默认实现中其实是返回true的, 在FrameLayout/LinearLayout等类中覆盖为false. 文档说默认为true的理由是兼容性, 不知道是为啥

判断是否是长按也用了一样的机制, 在ACTION_DOWN时, 如果不是在可滚动的parent中, 会postDelayed一个CheckForLongPress对象, 如果是在可滚动的parent中, 那么在CheckForTap.run触发的时候会postDelayed一个CheckForLongPress对象


第三:
在ACTION_UP事件中, 还会多考察一个focus状态, 基本的逻辑是:  

    boolean focusTaken = isFocusable() && isFocusableInTouchMode() && !isFocused() && requestFocus();
    if (!focusTaken) {
      post(mPerformClick) || performClickInternal();
    }

focusTaken只有在当前View是可focusableInTouchMode并且当前状态为非focused, 并且获取focus成功的情况下才是true

我们先拿三个常见的控件的默认情况来对比一下: 
  * TextView: 既不是focusable, 也不是focusableInTouchMode
  * Button: 是focusable, 但是不是focusableInTouchMode
  * EditText: 既是focusable, 也是focusableInTouchMode

那么:
  * TextView: isFocusableInTouchMode为false -> focusTaken = false
  * Button: isFocusableInTouchMode为false -> focusTaken = false;
  * EditText: isFocusableInTouchMode为true, 如果当前不是焦点, 那么EditText一般会成功获取焦点, focusTaken = true, 否则如果已经是焦点, 那么focusTaken = false

那么绝大多数情况下都会触发performClick  
只有EditText不一样, 给EditText设置一个OnClickListener的话, 第一次点击会获取焦点, 但是不触发点击事件, 第二次才会触发点击事件