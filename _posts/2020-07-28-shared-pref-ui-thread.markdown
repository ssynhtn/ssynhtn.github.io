---
layout: post
title:  "Does SharedPreferences load data in the UI thread"
date:   2020-07-28 22:10:26 +0800
categories: android
---
有一件事情我可能记错了，就是SharedPreferences第一次从文件中读取内容的是发生在哪个线程上

由于从没有看过spf的源码, so我是在某种XXX面试题中看到的spf的这类问题

不管如何，我的脑子里记得这样一件事情，就是spf第一次加载会阻塞主线程，后续读取因为内存中已经保存了内容，因此不会阻塞主线程

  

今天我发现实际上不是太正确，大概率是我记错了

实际上是这样的：

spf的实现类是SharedPreferencesImpl

它的构造函数的最后一行会调用startLoadFromDisk()，而这个函数的内容是：

    private void startLoadFromDisk() {  
	   synchronized (mLock) {  
	       mLoaded = false;  
	   }  
	   new Thread("SharedPreferencesImpl-load") {  
	       public void run() {  
	           loadFromDisk();  
	       }  
	   }.start();  
	}

  

so，这个加载是在一个后台线程中

but，getXX方法会等待这个初次加载完成

    public String getString(String key, @Nullable String defValue) {  
	    synchronized (mLock) {  
	        awaitLoadedLocked();  
	        String v = (String)mMap.get(key);  
	        return v != null ? v : defValue;  
	    }  
	}

  

    private void awaitLoadedLocked() {  
	    if (!mLoaded) {  
	        // Raise an explicit StrictMode onReadFromDisk for this  
	 // thread, since the real read will be in a different // thread and otherwise ignored by StrictMode.  BlockGuard.getThreadPolicy().onReadFromDisk();  
	    }  
	    while (!mLoaded) {  
	        try {  
	            mLock.wait();  
	        } catch (InterruptedException unused) {  
	        }  
	    }  
	    if (mThrowable != null) {  
	        throw new IllegalStateException(mThrowable);  
	    }  
	}

所以，基本上就相当是在主线程中读取了，并且没有办法直接绕过，毕竟配置内容本身是保存在硬盘中的，而几乎所有的app在启动时总是要读取用户是否已经登录的配置

but，我想了下在实际的app中还是有一个优化的余地的

因为在Application.onCreate中，我们有大量的第三方库的初始化，比如推送，统计，崩溃统计

有些初始化是可以放在后台线程/IntentService中的

有些则必须在主线程中第一时间初始化，比如崩溃统计（除非你有信心保证自己的app在初始化过程中没有bug，感觉我是没有这个信心）

而我这两天集成的友盟sdk，它声称可以在主线程中调用preInit来快速不阻塞地初始化，并在真正的初始化init放到后台线程中调用，可是实际上我测试这样做在它自己的集成测试中集成失败，并且preInit函数中forName了com.umeng.umzid.Spy类，这个类还在静态初始化块中加载了一个"umeng-spy"的native库，导致产生了文件读取的操作

因为可以很有信心地说，对大部分app， Application.onCreate方法肯定是有很多阻塞主线程的操作的，那么我们可以在Application.onCreate的最前面加上对SharedPreferences的获取，这样等到后面第一次真正去调用spf.getXX的时候，SharedPreferencesImpl中后台线程的loadFromDisk()就已经加载完成了或者完成一部分了