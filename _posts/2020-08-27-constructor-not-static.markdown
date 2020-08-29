---
layout: post
title:  "构造方法不是静态方法"
date:   2020-08-27 18:20:26 +0800
categories: java
---

不知道为什么总是记得在那里看到过说Java中的构造方法属于静态方法  
经过搜索发现是Thinking in Java这本当时没看完的书，当时初学Java的时候就觉得这个书很奇怪...

听上去貌似挺有道理到，因为在new Cat()的时候对象还不存在，那么应该类比于调用了一个静态方法创建了一个对象

但是从格式上来说构造方法是不用static来修饰的，也许是因为约定为程序员节约一点打字时间？
关于static修饰是否是为程序员省略了可以这样验证：

    public class CtorNotStatic {
        public static void main(String[] args) {
            try {
                Constructor<CtorNotStatic> ctor = CtorNotStatic.class.getDeclaredConstructor();
                boolean isStatic = Modifier.isStatic(ctor.getModifiers());
                boolean isOnlyPublic = ctor.getModifiers() == Modifier.PUBLIC;
                System.out.println("is constructor static: " + isStatic);
                System.out.println("is only public: " + isOnlyPublic);

            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
        }
    }

输出结果是，默认的构造方法的修饰符只有public, 没有static

构造方法中可以访问this，并且需要调用父类的构造方法，我们知道静态方法是没有覆盖一说的，因此不论怎么看构造方法如果一定要选择一个分类都应该是实例方法, 它是一个只能由虚拟机调用一次的特殊的初始化方法
这个方法的名字是<init>，这个在stacktrace中经常可以看到






