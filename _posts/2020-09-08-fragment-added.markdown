---
layout: post
title:  "Fragment怎么样才算是isAdded()"
date:   2020-09-08 18:21:26 +0800
categories: android
---

fragment manager的代码在不同版本之间似乎略有区别, 本记录查看的版本是:  
代码版本: androidx.fragment:fragment:1.1.0

某个版本把FragmentManagerImpl的代码直接剪切(喵喵喵?)到了FragmentManager中, 大概是放弃治疗了

Fragment.isAdded方法是这样的

    /**
     * Return true if the fragment is currently added to its activity.
     */
    final public boolean isAdded() {
        return mHost != null && mAdded;
    }

isVisible方法是这样的  

    /**
     * Return true if the fragment is currently visible to the user.  This means
     * it: (1) has been added, (2) has its view attached to the window, and
     * (3) is not hidden.
     */
    final public boolean isVisible() {
        return isAdded() && !isHidden() && mView != null
                && mView.getWindowToken() != null && mView.getVisibility() == View.VISIBLE;
    }

mHost的类型:

    // Host this fragment is attached to.
    FragmentHostCallback mHost;

FragmentHostCallback是一个抽象类, 它唯一的子类是FragmentActivity的内部类HostCallbacks  
所以mHost本质上指向activity对象

mHost变量的写操作全部在FragmentManagerImpl中发起

其中一个是调用了Fragment中的置空方法:

    /**
     * Called by the fragment manager once this fragment has been removed,
     * so that we don't have any left-over state if the application decides
     * to re-use the instance.  This only clears state that the framework
     * internally manages, not things the application sets.
     */
    void initState() {
        ...
        mHost = null;
        ...
    }

其它4个是FragmentManagerImpl类中直接赋值, 3个是赋值为FragmentManagerImpl的mHost对象, 一个是赋值为null  
而FragmentManagerImpl的mHost对象是在FragmentActivity的onCreate方法的第一行调用FragentController.attachHost获得  
在FragmentActivity的onDestroy方法中通过FragmentController.dispatchDestroy来置空

上面说的4个赋值地点  
其中两个是在onCreateView方法中  
另外两个是在moveToState(Fragment f, int newState, int transit, int transitionStyle, boolean keepActive)

FragmentManagerImpl类很神奇地实现了LayoutInflater.Factory2接口  
这和Fragment可以在xml中直接由&lt;fragment&gt;标签中inflate出来有关  
inflate出来的fragment会直接设置mHost为manager的mHost, 鉴于一般情况inflate方法是由FragmentActivity的子类在onCreate中调用的, 因此这个时候FragmentManagerImpl.mHost已经有值了  
这种情况暂时不是很感兴趣

moveToState方法会改变fragment当前的状态mState  
fragment有5个状态

    static final int INITIALIZING = 0;     // Not yet created.
    static final int CREATED = 1;          // Created.
    static final int ACTIVITY_CREATED = 2; // Fully created, not started.
    static final int STARTED = 3;          // Created and started, not resumed.
    static final int RESUMED = 4;          // Created started and resumed.

其中刚刚被new出来的时候状态是INITIALIZING, 这个名称看上去像是“正在初始化”, 但是这个名字其实好像不是很贴切  
比如说, 我们知道Java中每个new出来的对象是有一个初始化过程的, 但是初始化结束就结束了, 没有逆初始化或者重新初始化之类的过程  
但是这个mState在Fragment的方法中, 是可以在INITIALIZING和RESUMED之间来回变动的  
比如如果activity保持fragment对象的引用, 那么它可以在移除fragment后再添加fragment, 这样这个fragment就会在onDestroy后再次onCreate  
至于这种做法是否被推荐, 我猜测不是的, fragment作为activity的instance state的一部分, 在activity的旋转, 从后台恢复的过程中都会经历重新创建的过程  
如果偷看一下FragmentPagerAdapter的代码, 会发现fragment是通过detach来移除,  在添加的时候则会首先通过findFragmentByTag来查找. 

    performCreate(Bundle)
            mState = CREATED;
    performActivityCreated(Bundle)
            mState = ACTIVITY_CREATED;
    performStart()
            mState = STARTED;
    performResume()
            mState = RESUMED;
    performPause()
            mState = STARTED;
    performStop()
            mState = ACTIVITY_CREATED;
    performDestroyView()
            mState = CREATED;
    performDestroy()
            mState = INITIALIZING;


===吃午饭===

moveToState方法中给fragment.mHost赋值的地方, 一个是从INITIALIZING切换到任何其它状态的时候:
    if (f.mState <= newState) {
        switch (f.mState) {
            case Fragment.INITIALIZING:
                if (newState > Fragment.INITIALIZING) {
                    ...
                    f.mHost = mHost;

另一个是从其它状态切换到INITIALIZING状态的时候: 

    if (f.mState > newState) {
        switch (f.mState) {
                case Fragment.RESUMED:
                case Fragment.STARTED:  // fall through
                case Fragment.ACTIVITY_CREATED:
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (!keepActive) {
                                if (beingRemoved || mNonConfig.shouldDestroy(f)) {
                                    makeInactive(f);
                                } else {
                                    f.mHost = null;
                                    ...
                                }
                            }

其中切换到INITIALIZING时, 如果keepActive, 暂时不会置空mHost, 貌似keepActive和动画有关, 在动画结束后
会继续调用一个moveToState(fragment, fragment.getStateAfterAnimating(), 0, 0, false);

所以除了INITIALIZING这个状态以外, Fragment.mHost对象都是有的
如果查看对fragment进行操作的removeFragment和detachFragment这两个操作, 它们都会把fragment移除出FragmentManagerImpl.mAdded这个列表, 标记fragment.mAdded = false
区别是, 在这个fragment transaction中接下来会调用moveFragmentToExpectedState方法, 它会调用moveToState方法, 
对remove操作, newState传的是INITIALIZING, 对detach操作传的是fm.mState, 但是在moveToState方法内部又会检查f.mDetached, 如果是true, 那又设置newState为CREATED

好乱啊

但结果是, remove会让fragment重新进入INITIALIZING状态, fragment会经历onPause/Stop/DestroyView/Destroy, detach会进入CREATED状态, 不会走Destroy这一步, 但是会走onDestroyView

两个都是属于mAdded == false都情况

仔细看的话, Fragment.mAdded状态, 和fragment对象被加入fmimpl.mAdded这个list是一直保持一致的(但是remove的话还会从fmimpl.mActive这个map中移除, detach则不会)

那么基本的结论是, fragment的add/attach操作会让它进入added状态, 特点是fragment持有View对象  
remove/detach会让它added状态失效, 但是detach的时候f仍然保持着mHost引用, remove就没有了  