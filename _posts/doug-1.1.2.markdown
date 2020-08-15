---
layout: post
title:  "Doug Lea 1.1.2"
date:   2020-08-13 18:20:25 +0800
categories: java
---

1.1.2 Thread Mechanics

A thread is a call sequence that executes independently of others, while at the same time possibly
sharing underlying system resources such as files, as well as accessing other objects constructed
within the same program (see § 1.2.2). A java.lang.Thread object maintains bookkeeping and
control for this activity.

线程是一个调用序列
多个线程之间相互独立，但是同时与其它线程共享文件等系统资源，也可以访问其它线程构造的对象
Thread这个类是记录/控制真正的线程的

Every program consists of at least one thread — the one that runs the main method of the class
provided as a startup argument to the Java virtual machine ("JVM"). Other internal background
threads may also be started during JVM initialization. The number and nature of such threads vary
across JVM implementations. However, all user-level threads are explicitly constructed and started
from the main thread, or from any other threads that they in turn create.

理论上jvm只保证一个主线程
jvm的具体实现会加入很多其它线程
跑了下是：
1 threads
__Thread[main,5,main]   // 非守护线程
tg java.lang.ThreadGroup[name=system,maxpri=10]
3 threads // 3个都是守护线程
__Thread[Reference Handler,10,system]
__Thread[Finalizer,8,system]
__Thread[Signal Dispatcher,9,system]

Here is a summary of the principal methods and properties of class Thread, as well as a few usage
notes. They are further discussed and illustrated throughout this book. The Java™ Language
Specification ("JLS") and the published API documentation should be consulted for more detailed and
authoritative descriptions.


1.1.2.1 Construction
Different Thread constructors accept combinations of arguments supplying:
• A Runnable object, in which case a subsequent Thread.start invokes run of the
supplied Runnable object. If no Runnable is supplied, the default implementation of
Thread.run returns immediately.
• A String that serves as an identifier for the Thread. This can be useful for tracing and
debugging, but plays no other role.
• The ThreadGroup in which the new Thread should be placed. If access to the
ThreadGroup is not allowed, a SecurityException is thrown.
1.2版的Java，构造函数就3个参数：runnable, name, tg
如果传的tg为null，会获取当前线程的threadGroup

Class Thread itself implements Runnable. So, rather than supplying the code to be run in a
Runnable and using it as an argument to a Thread constructor, you can create a subclass of
Thread that overrides the run method to perform the desired actions. 
However, the best default
strategy is to define a Runnable as a separate class and supply it in a Thread constructor.
Isolating code within a distinct class relieves you of worrying about any potential interactions of
synchronized methods or blocks used in the Runnable with any that may be used by
methods of class Thread. More generally, this separation allows independent control over the nature
of the action and the context in which it is run: The same Runnable can be supplied to threads that
are otherwise initialized in different ways, as well as to other lightweight executors (see § 4.1.4). Also
note that subclassing Thread precludes a class from subclassing any other class.
不推荐扩展Thread类的理由，除了组合>继承之外
想不明白"any potential interactions of
synchronized methods or blocks used in the Runnable with any that may be used by
methods of class Thread."这个理由到底是什么，从安全性来说，在runnable中我也可以获取Thread.currentThread对象
Thread的实现中也没有public或者protected的field，因此本质上是一样的？

Thread objects also possess a daemon status attribute that cannot be set via any Thread
constructor, but may be set only before a Thread is started. The method setDaemon asserts that
the JVM may exit, abruptly terminating the thread, so long as all other non-daemon threads in the
program have terminated. The isDaemon method returns status. The utility of daemon status is very
limited. Even background threads often need to do some cleanup upon program exit. (The spelling of
daemon, often pronounced as "day-mon", is a relic of systems programming tradition. System
daemons are continuous processes, for example print-queue managers, that are "always" present on a
system.)
守护线程：jvm exit的时候可以直接停止该线程的执行
1， setDaemon方法只能在start之前调用


1.1.2.2 Starting threads
Invoking its start method causes an instance of class Thread to initiate its run method as an
independent activity. None of the synchronization locks held by the caller thread are held by the new
thread (see § 2.2.1).
A Thread terminates when its run method completes by either returning normally or throwing an
unchecked exception (i.e., RuntimeException, Error, or one of their subclasses). Threads
are not restartable, even after they terminate. Invoking start more than once results in an
InvalidThreadStateException.

The method isAlive returns true if a thread has been started but has not terminated. It will
return true if the thread is merely blocked in some way. JVM implementations have been known to
differ in the exact point at which isAlive returns false for threads that have been cancelled (see
§ 3.1.2). There is no method that tells you whether a thread that is not isAlive has ever been
started. Also, one thread cannot readily determine which other thread started it, although it may
determine the identities of other threads in its ThreadGroup (see § 1.1.2.6).

一个thread是无法判断是那个线程启动了他的, 不过它可以看到它所在的tg的其它线程（ThreadGroup.enumerate方法，这个方法的缺点是它返回的线程数被你穿进去的array长度限制，如果要获取全部就必须检查返回的index）:
    private static Thread[] getThreads(ThreadGroup tg) {
        Thread[] temp = new Thread[4];
        int end = tg.enumerate(temp, false);

        while (end == temp.length) {
            temp = new Thread[temp.length * 2];
            end = tg.enumerate(temp, false);
        }

        return Arrays.copyOf(temp, end);
    }


1.1.2.3 Priorities
To make it possible to implement the Java virtual machine across diverse hardware platforms and
operating systems, the Java programming language makes no promises about scheduling or fairness,
and does not even strictly guarantee that threads make forward progress (see § 3.4.1.5). But threads do
support priority methods that heuristically influence schedulers:
• Each Thread has a priority, ranging between Thread.MIN_PRIORITY and
Thread.MAX_PRIORITY (defined as 1 and 10 respectively).
• By default, each new thread has the same priority as the thread that created it. The initial
thread associated with a main by default has priority Thread.NORM_PRIORITY (5).
• The current priority of any thread can be accessed via method getPriority.
• The priority of any thread can be dynamically changed via method setPriority. The
maximum allowed priority for a thread is bounded by its ThreadGroup.
!!Java语言本身不保证线程能运行
线程的优先级也是一个hint

When more runnable (see § 1.3.2) threads than available CPUs, a scheduler is generally biased to
prefer running those with higher priorities. The exact policy may and does vary across platforms. For
example, some JVM implementations always select the thread with the highest current priority (with
ties broken arbitrarily). Some JVM implementations map the ten Thread priorities into a smaller
number of system-supported categories, so threads with different priorities may be treated equally.
And some mix declared priorities with aging schemes or other scheduling policies to ensure that even
low-priority threads eventually get a chance to run. Also, setting priorities may, but need not, affect
scheduling with respect to other programs running on the same computer system.
Priorities have no other bearing on semantics or correctness (see § 1.3). In particular, priority
manipulations cannot be used as a substitute for locking. Priorities can be used only to express the
relative importance or urgency of different threads, where these priority indications would be useful to
take into account when there is heavy contention among threads trying to get a chance to execute. For
example, setting the priorities of the particle animation threads in ParticleApplet below that of
the applet thread constructing them might on some systems improve responsiveness to mouse clicks,
and would at least not hurt responsiveness on others. But programs should be designed to run correctly
(although perhaps not as responsively) even if setPriority is defined as a no-op. (Similar
remarks hold for yield; see § 1.1.2.5.)

The following table gives one set of general conventions for linking task categories to priority
settings. In many concurrent applications, relatively few threads are actually runnable at any given
time (others are all blocked in some way), in which case there is little reason to manipulate priorities.
In other cases, minor tweaks in priority settings may play a small part in the final tuning of a
concurrent system.
Range Use
10 Crisis management
7-9 Interactive, event-driven
4-6 IO-bound
2-3 Background computation
1 Run only if nothing else can


1.1.2.4 Control methods
Only a few methods are available for communicating across threads:
• Each Thread has an associated boolean interruption status (see § 3.1.2). 
Invoking t.interrupt for some Thread t sets t's interruption status to true, unless
Thread t is engaged in Object.wait, Thread.sleep, or Thread.join; in
this case interrupt causes these actions (in t) to throw InterruptedException,
but t's interruption status is set to false.
• The interruption status of any Thread can be inspected using method isInterrupted.
This method returns true if the thread has been interrupted via the interrupt method
but the status has not since been reset either by the thread invoking
Thread.interrupted (see § 1.1.2.5) or in the course of wait, sleep, or join
throwing InterruptedException.
• Invoking t.join() for Thread t suspends the caller until the target Thread t
completes: the call to t.join() returns when t.isAlive() is false (see § 4.3.2).
A version with a (millisecond) time argument returns control even if the thread has not
completed within the specified time limit. Because of how isAlive is defined, it makes no
sense to invoke join on a thread that has not been started. For similar reasons, it is unwise
to try to join a Thread that you did not create.

Originally, class Thread supported the additional control methods suspend, resume, stop,
and destroy. Methods suspend, resume, and stop have since been deprecated; method
destroy has never been implemented in any release and probably never will be. The effects of
methods suspend and resume can be obtained more safely and reliably using the waiting and
notification techniques discussed in § 3.2. The problems surrounding stop are discussed in § 3.1.2.3.
suspend/resume被 wait/notify替代了
stop方法有潜在的问题
destroy一直没实现

1.1.2.5 Static methods
Some Thread class methods can be applied only to the thread that is currently running (i.e., the
thread making the call to the Thread method). To enforce this, these methods are declared as
static.
Thread的一些方法被定义成static，而不是让我们用Thread.currentThread().xxx()是有原因的，为了让这个方法只能被这个线程本身调用
• Thread.currentThread returns a reference to the current Thread. This reference
may then be used to invoke other (non-static) methods. For example,
Thread.currentThread().getPriority() returns the priority of the thread
making the call.
• Thread.interrupted clears interruption status of the current Thread and returns
previous status. (Thus, one Thread's interruption status cannot be cleared from other
threads.)
    public static boolean interrupted() {
        return currentThread().isInterrupted(true); // isInterrupted(boolean)是private方法
        // 因为我们只允许当前thread对自己清空中断状态
    }
    Thread还定义了一个非static的方法isInterrupted()，它不清空中断状态，所以可以被其它线程调用
• Thread.sleep(long msecs) causes the current thread to suspend for at least
msecs milliseconds (see § 3.2.2).
• Thread.yield is a purely heuristic hint advising the JVM that if there are any other
runnable but non-running threads, the scheduler should run one or more of these threads
rather than the current thread. The JVM may interpret this hint in any way it likes.


Despite the lack of guarantees, yield can be pragmatically effective on some single-CPU JVM
implementations that do not use time-sliced pre-emptive scheduling (see § 1.2.2). In this case, threads
are rescheduled only when one blocks (for example on IO, or via sleep). On these systems, threads
that perform time-consuming non-blocking computations can tie up a CPU for extended periods,
decreasing the responsiveness of an application. As a safeguard, methods performing non-blocking
computations that might exceed acceptable response times for event handlers or other reactive threads
can insert yields (or perhaps even sleeps) and, when desirable, also run at lower priority
settings. To minimize unnecessary impact, you can arrange to invoke yield only occasionally; for
example, a loop might contain:
if (Math.random() < 0.01) Thread.yield();
On JVM implementations that employ pre-emptive scheduling policies, especially those on
multiprocessors, it is possible and even desirable that the scheduler will simply ignore this hint
provided by yield.
yield方法可能在非抢占式调度的cpu中有帮助，那种系统中，一个线程如果没有阻塞就不会放弃cpu，容易造成其它线程长时间无法运行
如果ui线程不运行，app就会卡顿


1.1.2.6 ThreadGroups
Every Thread is constructed as a member of a ThreadGroup, by default the same group as that
of the Thread issuing the constructor for it. ThreadGroups nest in a tree-like fashion. When an
object constructs a new ThreadGroup, it is nested under its current group. The method
getThreadGroup returns the group of any thread. The ThreadGroup class in turn supports
methods such as enumerate that indicate which threads are currently in the group.

ThreadGroup.enumerate可以帮助获取tg中的所有线程

One purpose of class ThreadGroup is to support security policies that dynamically restrict access
to Thread operations; for example, to make it illegal to interrupt a thread that is not in your
group. This is one part of a set of protective measures against problems that could occur, for example,
if an applet were to try to kill the main screen display update thread. ThreadGroups may also
place a ceiling on the maximum priority that any member thread can possess.
线程组的目的的可以动态改变安全策略，例如禁止组外的线程interrupt组内的线程
默认的checkAccess方法只对根线程组进行保护：
    public void checkAccess(Thread t) {
        if (t == null) {
            throw new NullPointerException("thread can't be null");
        }
        if (t.getThreadGroup() == rootGroup) {
            checkPermission(SecurityConstants.MODIFY_THREAD_PERMISSION);
        } else {
            // 非根线程组的线程直接通过
            // just return
        }
    }
但是SecurityManager的子类AppletSecurity有自定义的实现，里面似乎比较乱
    public void checkAccess(Thread var1) {
        if (var1.getState() != State.TERMINATED && !this.inThreadGroup(var1)) {
            this.checkPermission(SecurityConstants.MODIFY_THREAD_PERMISSION);
        }

    }

ThreadGroups tend not to be used directly in thread-based programs. In most applications,
normal collection classes (for example java.util.Vector) are better choices for tracking
groups of Thread objects for application-dependent purposes.

Among the few ThreadGroup methods that commonly come into play in concurrent programs is
method uncaughtException, which is invoked when a thread in a group terminates due to an
uncaught unchecked exception (for example a NullPointerException). This method
normally causes a stack trace to be printed。

ThreadGroup实现了uncaughtExceptionHandler接口
未捕获的exception的处理是从Thread.dispatchUncaughtException这个函数开始的
1, 一个线程可以给它直接设置一个uncaughtExceptionHandler（几乎没看到有人这么做），那么这里处理
2，给它的threadGroup处理

ThreadGroup默认的处理方式：
1，如果不是root group，就传给parent
2，是root，如果getDefaultUncaughtExceptionHandler() != null, 这里处理， 否则就往stderr打印一下线程的name和stacktrace

通过设置DefaultUncaughtExceptionHandler可以记录bug
