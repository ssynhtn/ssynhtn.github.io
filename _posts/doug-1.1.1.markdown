---
layout: post
title:  "Doug Lea 1.1.1"
date:   2020-08-08 18:20:25 +0800
categories: java
---

1.1 Using Concurrency Constructs
使用并发构造？

This section introduces basic concurrency support constructs by example and then proceeds with awalk-through of the principal methods of class Thread. Other concurrency constructs are briefly
described as they are introduced, but full technical details are postponed to later chapters (mainly §2.2.1 and § 3.2.2). Also, concurrent programs often make use of a few ordinary Java programminglanguage features that are not as widely used elsewhere. These are briefly reviewed as they arise.

基本的支持并发编程的构造
Thread类的主要方法


1.1.1 A Particle Applet
粒子applet
ParticleApplet is an Applet that displays randomly moving particles. In addition to concurrency constructs, this example illustrates a few of the issues encountered when using threads with any GUI-based program. The version described here needs a lot of embellishment to be visually attractive or realistic. You might enjoy experimenting with additions and variations as an exercise.
ParticleApplet类
图形界面编程中使用线程都会碰到的一些问题

As is typical of GUI-based programs, ParticleApplet uses several auxiliary classes that do most of the work. We'll step through construction of the Particle and ParticleCanvas classes before discussing ParticleApplet.
applet依赖于Paticle, ParticleCancas这两个类

1.1.1.1 Particle
The Particle class defines a completely unrealistic model of movable bodies. Each particle isrepresented only by its (x, y) location. Each particle also supports a method to randomly change itslocation and a method to draw itself (as a small square) given a supplied java.awt.Graphicsobject.

While Particle objects do not themselves exhibit any intrinsic concurrency, their methods may beinvoked across multiple concurrent activities. When one activity is performing a move and another isinvoking draw at about the same time, we'd like to make sure that the draw paints an accuraterepresentation of where the Particle is. Here, we require that draw uses the location valuescurrent either before or after the move. For example, it would be conceptually wrong for a drawoperation to display using the y-value current before a given move, but the x-value current after themove. If we were to allow this, then the draw method would sometimes display the particle at alocation that it never actually occupied.

Particle类有move， draw两个方法
如果不同的线程调用这2个方法，可能出现的错误：draw的时候x，y取值，一个为move之前的，一个为之后的


This protection can be obtained using the synchronized keyword, which can modify either a method or a block of code. Every instance of class Object (and its subclasses) possesses a lock that is obtained on entry to a synchronized method and automatically released upon exit. The code-block version works in the same way except that it takes an argument stating which object to lock. The most common argument is this, meaning to lock the object whose method is executing. When alock is held by one thread, other threads must block waiting for the holding thread to release the lock.Locking has no effect on non-synchronized methods, which can execute even if the lock is being heldby another thread

synchronized关键字可以防止这种情况发生
Java中每个对象都拥有一个锁，一个线程进入一个synchronized方法时会获取这个锁(锁住当前对象)，或者进入一个synchronized block(同步块)时获取同步块参数的锁. 当锁被一个线程持有，其它线程进入同步块时会被阻塞(block). 不过锁对非同步对代码块没有这种作用.

Locking provides protection against both high-level and low-level conflicts by enforcing atomicity among methods and code-blocks synchronized on the same object.
被同步对代码块和方法，锁强制了原子性, 防止了包括高层次和低层次的冲突 

Atomic actions are performed as units, without any interleaving of the actions of other threads. But, as discussed in §1.3.2 and in Chapter 2, too much locking can also produce liveness problems that cause programs tofreeze up. Rather than exploring these issues in detail now, we'll rely on some simple default rules for writing methods that preclude interference problems:
• Always lock during updates to object fields.
• Always lock during access of possibly updated object fields.
• Never lock when invoking methods on other objects.

太多锁会造成活跃性问题
三个基本的原则：
    改动字段时要锁
    获取字段也要锁？
    调用其它对象的方法，不要锁

These rules have many exceptions and refinements, but they provide enough guidance to write class Particle
这些原则不是死的，但是基本上是好的

class Particle {protected int x;protected int y;protected final Random rng = new Random();public Particle(int initialX, int initialY) {x = initialX;y = initialY;}public synchronized void move() {x += rng.nextInt(10) - 5;y += rng.nextInt(20) - 10;}public void draw(Graphics g) {int lx, ly;synchronized (this) { lx = x; ly = y; }g.drawRect(lx, ly, 10, 10);}}


The use of final in the declaration of the random number generator rng reflects ourdecision that this reference field cannot be changed, so it is not impacted by our locking rules.Many concurrent programs use final extensively, in part as helpful, automatically enforced documentation of design decisions that reduce the need for synchronization (see §2.1).
final在这里有任何好处吗？
• The draw method needs to obtain a consistent snapshot of both the x and y values. Since asingle method can return only one value at a time, and we need both the x and y values here,we cannot easily encapsulate the field accesses as a synchronized method. We insteaduse a synchronized block. (See § 2.4 for some alternatives.)
为了获取一致的x, y的数值，用了一个同步块获取
• The draw method conforms to our rule of thumb to release locks during method invocations on other objects (here g.drawRect). The move method appears to break this rule bycalling rng.nextInt. However, this is a reasonable choice here because each Particle confines its own rng — conceptually, the rng is just a part of theParticle itself, so it doesn't count as an "other" object in the rule. Section § 2.3 describes more general conditions under which this sort of reasoning applies and discusses factors thatshould be taken into account to be sure that this decision is warranted
draw方法跟随了原则，在同步块外部调用canvas的draw方法
move方法似乎是违反了这个原则，但是rnd这个对象是我们自己拥有的对象，所以它不算“其它”对象?


1.1.1.2 ParticleCanvas
ParticleCanvas is a simple subclass of java.awt.Canvas that provides a drawing area for all of the Particles. Its main responsibility is to invoke draw for all existing particles whenever its paint method is called.
However, the ParticleCanvas itself does not create or manage the particles. It needs either tobe told about them or to ask about them. Here, we choose the former.
粒子画布类 extends java.awt.Canvas，它不管理粒子对象，只负责画画
要么它主动获取粒子，要么我们注入粒子，这里我们选择向它注入粒子对象

class ParticleCanvas extends Canvas {private Particle[] particles = new Particle[0];ParticleCanvas(int size) {setSize(new Dimension(size, size));}// intended to be called by appletprotected synchronized void setParticles(Particle[] ps) {if (ps == null)throw new IllegalArgumentException("Cannot set null");particles = ps;}protected synchronized Particle[] getParticles() {return particles;}public void paint(Graphics g) { // override Canvas.paintParticle[] ps = getParticles();for (int i = 0; i < ps.length; ++i)ps[i].draw(g);}}

The Particle and ParticleCanvas classes could be used as the basis of several differentprograms. But in ParticleApplet all we want to do is set each of a collection of particles in autonomous "continuous" motion and update the display accordingly to show where they are. Tocomply with standard applet conventions, these activities should begin when Applet.start is externally invoked (normally from within a web browser), and should end when Applet.stop isinvoked. (We could also add buttons allowing users to start and stop the particle animation themselves.)
我们的目标是让粒子自主，连续地移动

There are several ways to implement all this. Among the simplest is to associate an independent loopwith each particle and to run each looping action in a different thread.
一个傻瓜方法是：每个粒子由一个线程在一个循环中控制

Each instance of the Thread class maintains the control state necessary to execute and manage thecall sequence comprising its action. The most commonly used constructor in class Thread accepts aRunnable object as an argument, which arranges to invoke the Runnable's run method whenthe thread is started. While any class can implement Runnable, it often turns out to be bothconvenient and helpful to define a Runnable as an anonymous inner class


The ParticleApplet class uses threads in this way to put particles into motion, and cancelsthem when the applet is finished. This is done by overriding the standard Applet methods start
and stop (which have the same names as, but are unrelated to, methods Thread.start andThread.stop).
Thread.stop方法已经废弃了, 实际用interrupt方法来通知线程，线程中的代码主动检查interrupt来判断是否结束运行

The above interaction diagram shows the main message sequences during execution of the applet. 
Applet.start -> Thread.start -> run -> particle.move, canvas.repaint -> 发送awt event
awt线程处理awt事件：canvas.paint -> particle.draw

In addition to the threads explicitly created, this applet interacts with the AWT event thread, described in
more detail in § 4.1.4. The producer-consumer relationship extending from the omitted right hand side
of the interaction diagram takes the approximate form:
大致是应用线程（非ui线程）向awt线程发送event，在awt线程中处理event，运行应用代码

拷贝一下代码
class Particle {
protected int x;
protected int y;
protected final Random rng = new Random();
public Particle(int initialX, int initialY) {
x = initialX;
y = initialY;
}
public synchronized void move() {
x += rng.nextInt(10) - 5;
y += rng.nextInt(20) - 10;
}
public void draw(Graphics g) {
int lx, ly;
synchronized (this) { lx = x; ly = y; }
g.drawRect(lx, ly, 10, 10);
}
}
class ParticleCanvas extends Canvas {
private Particle[] particles = new Particle[0];
ParticleCanvas(int size) {
setSize(new Dimension(size, size));
}
// intended to be called by applet
protected synchronized void setParticles(Particle[] ps) {
if (ps == null)
throw new IllegalArgumentException("Cannot set null");
particles = ps;
}
protected synchronized Particle[] getParticles() {
return particles;
}
public void paint(Graphics g) { // override Canvas.paint
Particle[] ps = getParticles();
for (int i = 0; i < ps.length; ++i)
ps[i].draw(g);
}
}
public class ParticleApplet extends Applet {
protected Thread[] threads = null; // null when not running
protected final ParticleCanvas canvas
= new ParticleCanvas(100);
public void init() { add(canvas); }
protected Thread makeThread(final Particle p) { // utility
Runnable runloop = new Runnable() {
public void run() {
try {
for(;;) {   // 这里如果检查一下当前线程是否被中断会更好
p.move();
canvas.repaint();
Thread.sleep(100); // 100msec is arbitrary
}
}
catch (InterruptedException e) { return; }
}
};
return new Thread(runloop);
}
public synchronized void start() {
int n = 10; // just for demo
if (threads == null) { // bypass if already started
Particle[] particles = new Particle[n];
for (int i = 0; i < n; ++i)
particles[i] = new Particle(50, 50);
canvas.setParticles(particles);
threads = new Thread[n];
for (int i = 0; i < n; ++i) {
threads[i] = makeThread(particles[i]);
threads[i].start();
}
}
}
public synchronized void stop() {
if (threads != null) { // bypass if already stopped
for (int i = 0; i < threads.length; ++i)
threads[i].interrupt();
threads = null;
}
}
}

代码备注:



The use of final in the declaration of the random number generator rng reflects our
decision that this reference field cannot be changed, so it is not impacted by our locking rules.
Many concurrent programs use final extensively, in part as helpful, automatically
enforced documentation of design decisions that reduce the need for synchronization (see §
2.1).
使用final的字段，属于代码即文档，访问final字段不需要同步

• The draw method needs to obtain a consistent snapshot of both the x and y values. Since a
single method can return only one value at a time, and we need both the x and y values here,
we cannot easily encapsulate the field accesses as a synchronized method. We instead
use a synchronized block. (See § 2.4 for some alternatives.)
draw方法为了一次性获取x和y的值，使用了同步块

• The draw method conforms to our rule of thumb to release locks during method invocations
on other objects (here g.drawRect). 

The move method appears to break this rule by
calling rng.nextInt. However, this is a reasonable choice here because each
Particle confines its own rng — conceptually, the rng is just a part of the
Particle itself, so it doesn't count as an "other" object in the rule. Section § 2.3 describes
more general conditions under which this sort of reasoning applies and discusses factors that
should be taken into account to be sure that this decision is warranted

draw方法遵从了之前的原则：调用其它对象的方法要在同步块以外
move方法貌似违反了这个原则，但是rng对象似乎可以看成是this的一部分


The action in makeThread defines a "forever" loop (which some people prefer to write
equivalently as "while (true)") that is broken only when the current thread is
interrupted. During each iteration, the particle moves, tells the canvas to repaint so the move
will be displayed, and then does nothing for a while, to slow things down to a humanviewable
rate. Thread.sleep pauses the current thread. It is later resumed by a system
timer.
• One reason that inner classes are convenient and useful is that they capture all appropriate
context variables — here p and canvas — without the need to create a separate class with
fields that record these values. This convenience comes at the price of one minor
awkwardness: All captured method arguments and local variables must be declared as
final, as a guarantee that the values can indeed be captured unambiguously. Otherwise, for
example, if p were reassigned after constructing the Runnable inside method
makeThread, then it would be ambiguous whether to use the original or the assigned
value when executing the Runnable.
• The call to canvas.repaint does not directly invoke canvas.paint. The
repaint method instead places an UpdateEvent on a java.awt.EventQueue.
repaint方法不会直接去调用paint(那样会直接在当前线程去paint)，而是往EventQueue放了一个UpdateEvent，这个类找不到啊？1.2里似乎也没有，只有PaintEvent
(This may be internally optimized and further manipulated to eliminate duplicate events.) A
java.awt.EventDispatchThread asynchronously takes this event from the queue
and dispatches it by (ultimately) invoking canvas.paint. This thread and possibly other
system-created threads may exist even in nominally single-threaded programs.
• The activity represented by a constructed Thread object does not begin until invocation of
the Thread.start method.
• As discussed in § 3.1.2, there are several ways to cause a thread's activity to stop. The
simplest is just to have the run method terminate normally. But in infinitely looping
methods, the best option is to use Thread.interrupt. An interrupted thread will
automatically abort (via an InterruptedException) from the methods
Object.wait, Thread.join, and Thread.sleep. Callers can then catch this
exception and take any appropriate action to shut down. Here, the catch in runloop just
causes the run method to exit, which in turn causes the thread to terminate.
• The start and stop methods are synchronized to preclude concurrent starts or
stops. Locking works out OK here even though these methods need to perform many
operations (including calls to other objects) to achieve the required started-to-stopped or
stopped-to-started state transitions. Nullness of variable threads is used as a convenient
state indicator
让一个线程结束的最简单的方法是使用中断interrupt()函数
当一个线程处于阻塞状态时：当它在调用 someObject.wait(), otherThread.join(), Thread.sleep()时