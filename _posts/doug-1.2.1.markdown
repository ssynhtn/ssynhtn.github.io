---
layout: post
title:  "Doug Lea 1.2.1"
date:   2020-08-19 18:20:25 +0800
categories: java
---

1.2 Objects and Concurrency
There are many ways to characterize objects, concurrency, and their relationships. This section
discusses several different perspectives — definitional, system-based, stylistic, and modeling-based —
that together help establish a conceptual basis for concurrent object-oriented programming
面向对象/并发，有很多不同的角度去看待
definitional, 定义上的
system-based, 基于系统的？
stylistic,  文体的？
modeling-based： 基于模型的？

1.2.1 Concurrency
Like most computing terms, "concurrency" is tricky to pin down. Informally, a concurrent program is
one that does more than one thing at a time. For example, a web browser may be simultaneously
performing an HTTP GET request to get an HTML page, playing an audio clip, displaying the number
of bytes received of some image, and engaging in an advisory dialog with a user. However, this
simultaneity is sometimes an illusion. On some computer systems these different activities might
indeed be performed by different CPUs. But on other systems they are all performed by a single timeshared
CPU that switches among different activities quickly enough that they appear to be
simultaneous, or at least nondeterministically interleaved, to human observers.
并发大致上可以定义为：如果一个程序可以同时执行多个任务，那么这个程序可以成为并发程序
但是这个同时执行可能只是一个表象（尤其是单核/单cpu的机器上）：一个cpu可以是分时的，它快速在多个任务之间切换，当人类观看时，一个短时间内cpu已经执行多次切换，人类无法区分，就会误认为是任务是同时执行/发生

A more precise, though not very interesting definition of concurrent programming can be phrased
operationally: A Java virtual machine and its underlying operating system (OS) provide mappings
from apparent simultaneity to physical parallelism (via multiple CPUs), or lack thereof, by allowing
independent activities to proceed in parallel when possible and desirable, and otherwise by timesharing.
Concurrent programming consists of using programming constructs that are mapped in this
way. Concurrent programming in the Java programming language entails using Java programming
language constructs to this effect, as opposed to system-level constructs that are used to create new
operating system processes. By convention, this notion is further restricted to constructs affecting a
single JVM, as opposed to distributed programming, for example using remote method invocation
(RMI), that involves multiple JVMs residing on multiple computer systems.
一个更准确一点的定义是：虚拟机/操作系统将我们想要同时运行的代码分配给不同的cpu同时运行，当cpu数不够时则让cpu分时运行不同的代码

Concurrency and the reasons for employing it are better captured by considering the nature of a few
common types of concurrent applications:
直接看并发的应用更能直观地理解它并明白使用它的原因：

Web services. Most socket-based web services (for example, HTTP daemons, servlet engines, and
application servers) are multithreaded. Usually, the main motivation for supporting multiple
concurrent connections is to ensure that new incoming connections do not need to wait out completion
of others. This generally minimizes service latencies and improves availability.
基于socket的网络服务

Number crunching. Many computation-intensive tasks can be parallelized, and thus execute more
quickly if multiple CPUs are present. Here the goal is to maximize throughput by exploiting
parallelism.    
数字运算

I/O processing. Even on a nominally sequential computer, devices that perform reads and writes on
disks, wires, etc., operate independently of the CPU. Concurrent programs can use the time otherwise
wasted waiting for slow I/O, and can thus make more efficient use of a computer's resources.
一个程序等待io操作的时候，cpu可以去运行其它的任务

Simulation. Concurrent programs can simulate physical objects with independent autonomous
behaviors that are hard to capture in purely sequential programs.
模拟：用一个线程模拟一个物体可以优雅地解决模拟的问题：用同步代码根本难以做到

GUI-based applications. Even though most user interfaces are intentionally single-threaded, they
often establish or communicate with multithreaded services. Concurrency enables user controls to stay
responsive even during time-consuming actions.
图形界面：大部分gui都是刻意设计成单线程的(事件驱动的)，但是但应用在进行耗时的操作时，并发使得UI可以和用户保持迅速流畅的交互

Component-based software. Large-granularity software components (for example those providing
design tools such as layout editors) may internally construct threads in order to assist in bookkeeping,
provide multimedia support, achieve greater autonomy, or improve performance.

Mobile code. Frameworks such as the java.applet package execute downloaded code in
separate threads as one part of a set of policies that help to isolate, monitor, and control the effects of
unknown code.
applet在单独的线程中运行下载的代码

Embedded systems. Most programs running on small dedicated devices perform real-time control.
Multiple components each continuously react to external inputs from sensors or other devices, and
produce external outputs in a timely manner. As defined in The Java™ Language Specification, the
Java platform does not support hard real-time control in which system correctness depends on actions
being performed by certain deadlines. Particular run-time systems may provide the stronger
guarantees required in some safety-critical hard-real-time systems. But all JVM implementations
support soft real-time control, in which timeliness and performance are considered quality-of-service
issues rather than correctness issues (see § 1.3.3). This reflects portability goals that enable the JVM to
be implemented on modern opportunistic, multipurpose hardware and system software.



1.2.2 Concurrent Execution Constructs
Threads are only one of several constructs available for concurrently executing code. 
The idea of generating a new activity can be mapped to any of several abstractions along a granularity continuum reflecting trade-offs of autonomy versus overhead. Thread-based designs do not always provide the
best solution to a given concurrency problem. Selection of one of the alternatives discussed below can
provide either more or less security, protection, fault-tolerance, and administrative control, with either
more or less associated overhead. Differences among these options (and their associated programming
support constructs) impact design strategies more than do any of the details surrounding each one.
实现并发的构造有好几种，线程只是其中的一种, 它们在粒度上不同，每一种都在自治性和开销上做了一个折中

1.2.2.1 Computer systems
1.2.2.2 Processes
1.2.2.3 Threads
1.2.2.4 Tasks and lightweight executable frameworks

If you had a large supply of computer systems, you might map each logical unit of execution to a
different computer. Each computer system may be a uniprocessor, a multiprocessor, or even a cluster
of machines administered as a single unit and sharing a common operating system. This provides
unbounded autonomy and independence. Each system can be administered and controlled separately
from all the others.
However, constructing, locating, reclaiming, and passing messages among such entities can be
expensive, opportunities for sharing local resources are eliminated, and solutions to problems
surrounding naming, security, fault-tolerance, recovery, and reachability are all relatively heavy in
comparison with those seen in concurrent programs. So this mapping choice is typically applied only
for those aspects of a system that intrinsically require a distributed solution. And even here, all but the
tiniest embedded computer devices host more than one process.
一台计算机当然可以作为一个并发单元（而且这可以保证真正的并发）
但是它问题很多：为每台计算机指定名字，安全，计算机之间的通信等

A process is an operating-system abstraction that allows one computer system to support many units
of execution. Each process typically represents a separate running program; for example, an executing
JVM. Like the notion of a "computer system", a "process" is a logical abstraction, not a physical one.
So, for example, bindings from processes to CPUs may vary dynamically.
进程是操作系统上的一个抽象，它允许一个系统执行多个任务，注意到执行一个进程的cpu是可以动态变化的

Operating systems guarantee some degree of independence, lack of interference, and security among
concurrently executing processes. Processes are generally not allowed to access one another's storage
locations (although there are usually some exceptions), and must instead communicate via
interprocess communication facilities such as pipes. 
操作系统保证进程之间的独立性, 绝大部分情况下，进程之间不允许相互访问数据
它们会使用ipc机制，例如管道？
Most systems make at least best-effort promises
about how processes will be created and scheduled. This nearly always entails pre-emptive timeslicing
— suspending processes on a periodic basis to give other processes a chance to run.
大部分系统都会对进程的执行做出一个保证，这基本上就意味着这个系统使用抢占式分时调度

The overhead for creating, managing, and communicating among processes can be a lot lower than in
per-machine solutions. However, since processes share underlying computational resources (CPUs,
memory, IO channels, and so on), they are less autonomous. For example, a machine crash caused by
one process kills all processes.
进程之间的沟通成本相对更低，但是自治性也降低了

Thread constructs of various forms make further trade-offs in autonomy, in part for the sake of lower
overhead. The main trade-offs are:
线程比进程更轻
Sharing. Threads may share access to the memory, open files, and other resources associated with a
single process. Threads in the Java programming language may share all such resources. Some
operating systems also support intermediate constructions, for example "lightweight processes" and
"kernel threads" that share only some resources, do so only upon explicit request, or impose other
restrictions.
不同线程共享内存，文件

Scheduling. Independence guarantees may be weakened to support cheaper scheduling policies. At
one extreme, all threads can be treated together as a single-threaded process, in which case they may
cooperatively contend with each other so that only one thread is running at a time, without giving any
other thread a chance to run until it blocks (see § 1.3.2). At the other extreme, the underlying
scheduler can allow all threads in a system to contend directly with each other via pre-emptive
scheduling rules. Threads in the Java programming language may be scheduled using any policy lying
at or anywhere between these extremes.
线程调度的自由度很高，在一个极端系统可以只允许进程执行一个线程，在另一个极端，它可以允许所有进程中的线程一起竞争cpu


Communication. Systems interact via communication across wires or wireless channels, for example
using sockets. Processes may also communicate in this fashion, but may also use lighter mechanisms
such as pipes and interprocess signalling facilities. Threads can use all of these options, plus other
cheaper strategies relying on access to memory locations accessible across multiple threads, and
employing memory-based synchronization facilities such as locks and waiting and notification
mechanisms. These constructs support more efficient communication, but sometimes incur the
expense of greater complexity and consequently greater potential for programming error
系统之间用socket进行通信，进程之间也是可以这样做的，但是更可以使用更轻量级的通信方式：比如管道和信号
线程就更更直接了，可以用内存地址（对象的引用）直接交换数据，但是这样我们需要使用锁等机制来同步，这大大增加编程难度，容易造成编程错误


The trade-offs made in supporting threads cover a wide range of applications, but are not always
perfectly matched to the needs of a given activity. While performance details differ across platforms,
the overhead in creating a thread is still significantly greater than the cheapest (but least independent)
way to invoke a block of code — calling it directly in the current thread.
When thread creation and management overhead become performance concerns, you may be able to
make additional trade-offs in autonomy by creating your own lighter-weight execution frameworks
that impose further restrictions on usage (for example by forbidding use of certain forms of blocking),
or make fewer scheduling guarantees, or restrict synchronization and communication to a more
limited set of choices. As discussed in § 4.1.4, these tasks can then be mapped to threads in about the
same way that threads are mapped to processes and computer systems.
The most familiar lightweight executable frameworks are event-based systems and subsystems (see §
1.2.3, § 3.6.4, and § 4.1), in which calls triggering conceptually asynchronous activities are
maintained as events that may be queued and processed by background threads. Several additional
lightweight executable frameworks are described in Chapter 4. When they apply, construction and use
of such frameworks can improve both the structure and performance of concurrent programs. Their
use reduces concerns (see § 1.3.3) that can otherwise inhibit the use of concurrent execution
techniques for expressing logically asynchronous activities and logically autonomous objects (see §
1.2.4).
基于事件队列的系统（awt event queue， looper/handler）比线程更简单（不会如多线程容易出错），也没有新建线程的开销


1.2.3 Concurrency and OO Programming

Objects and concurrency have been linked since the earliest days of each. The first concurrent OO
programming language (created circa 1966), Simula, was also the first OO language, and was among
the first concurrent languages. Simula's initial OO and concurrency constructs were somewhat
primitive and awkward. For example, concurrency was based around coroutines — thread-like
constructs requiring that programmers explicitly hand off control from one task to another. Several
other languages providing both concurrency and OO constructs followed — indeed, even some of the
earliest prototype versions of C++ included a few concurrency-support library classes. And Ada
(although, in its first versions, scarcely an OO language) helped bring concurrent programming out
from the world of specialized, low-level languages and systems.
Simula是第一个面向对象的支持并发的语言，其实也是第一个面向对象的语言
它的并发构造是coroutines，协程，和kotlin一样？不过Lea觉得它灰常尴尬

OO design played no practical role in the multithreaded systems programming practices emerging in
the 1970s. And concurrency played no practical role in the wide-scale embrace of OO programming
that began in the 1980s. But interest in OO concurrency stayed alive in research laboratories and
advanced development groups, and has re-emerged as an essential aspect of programming in part due
to the popularity and ubiquity of the Java platform.
面向对象的并发编程的流行是Java的功劳
Concurrent OO programming shares most features with programming of any kind. But it differs in
critical ways from the kinds of programming you may be most familiar with, as discussed below.

面向对象的并发编程和其它类型的编程的区别：

1.2.3.1 Sequential OO programming
Concurrent OO programs are often structured using the same programming techniques and design
patterns as sequential OO programs (see for example § 1.4). But they are intrinsically more complex.
When more than one activity can occur at a time, program execution is necessarily nondeterministic.
Code may execute in surprising orders — any order that is not explicitly ruled out is allowed (see §
2.2.7). So you cannot always understand concurrent programs by sequentially reading through their
code. For example, without further precautions, a field set to one value in one line of code may have a
different value (due to the actions of some other concurrent activity) by the time the next line of code
is executed. Dealing with this and other forms of interference often introduces the need for a bit more
rigor and a more conservative outlook on design.
并发编程的难点是你难以通过顺序阅读代码来理解它的运行过程
1.2.3.2 Event-based programming
Some concurrent programming techniques have much in common with those in event frameworks
employed in GUI toolkits supported by java.awt and javax.swing, and in other languages
such as Tcl/Tk and Visual Basic. In GUI frameworks, events such as mouse clicks are encapsulated as
Event objects that are placed in a single EventQueue. These events are then dispatched and
processed one-by-one in a single event loop, which normally runs as a separate thread. As discussed in
§ 4.1, this design can be extended to support additional concurrency by (among other tactics) creating
multiple event loop threads, each concurrently processing events, or even dispatching each event in a
separate thread. Again, this opens up new design possibilities, but also introduces new concerns about
interference and coordination among concurrent activities.

1.2.3.3 Concurrent systems programming
Object-oriented concurrent programming differs from multithreaded systems programming in
languages such as C mainly due to the encapsulation, modularity, extensibility, security, and safety
features otherwise lacking in C. Additionally, concurrency support is built into the Java programming
language, rather than supplied by libraries. This eliminates the possibility of some common errors, and
also enables compilers to automatically and safely perform some optimizations that would need to be
performed manually in C.
While concurrency support constructs in the Java programming language are generally similar to those
in the standard POSIX pthreads library and related packages typically used in C, there are some
important differences, especially in the details of waiting and notification (see § 3.2.2). It is very
possible to use utility classes that act almost just like POSIX routines (see § 3.4.4). But it is often
more productive instead to make minor adjustments to exploit the versions that the language directly
supports.

1.2.3.4 Other concurrent programming languages
Essentially all concurrent programming languages are, at some level, equivalent, if only in the sense
that all concurrent languages are widely believed not to have defined the right concurrency features.
However, it is not all that hard to make programs in one language look almost equivalent to those in
other languages or those using other constructs, by developing packages, classes, utilities, tools, and
coding conventions that mimic features built into others. In the course of this book, constructions are
introduced that provide the capabilities and programming styles of semaphore-based systems (§ 3.4.1),
futures (§ 4.3.3), barrier-based parallelism (§ 4.4.3), CSP (§ 4.5.1) and others. It is a perfectly great
idea to write programs using only one of these styles, if this suits your needs. However, many
concurrent designs, patterns, frameworks, and systems have eclectic heritages and steal good ideas
from anywhere they can.
信号量
future
屏障
csp(通信顺序进程？)


1.2.4 Object Models and Mappings
Conceptions of objects often differ across sequential versus concurrent OO programming, and even
across different styles of concurrent OO programming. Contemplation of the underlying object models
and mappings can reveal the nature of differences among programming styles hinted at in the previous
section.


1.2.4.1 Object models
The WaterTank class uses objects to model reality. Object models provide rules and frameworks
for defining objects more generally, covering:
Statics. The structure of each object is described (normally via a class) in terms of internal attributes
(state), connections to other objects, local (internal) methods, and methods or ports for accepting
messages from other objects.
Encapsulation. Objects have membranes separating their insides and outsides. Internal state can be
directly modified only by the object itself. (We ignore for now language features that allow this rule to
be broken.)
Communication. Objects communicate only via message passing. Objects issue messages that trigger
actions in other objects. The forms of these messages may range from simple procedural calls to those
transported via arbitrary communication protocols.
Identity. New objects can be constructed at any time (subject to system resource constraints) by any
object (subject to access control). Once constructed, each object maintains a unique identity that
persists over its lifetime.
Connections. One object can send messages to others if it knows their identities. Some models rely on
channel identities rather than or in addition to object identities. Abstractly, a channel is a vehicle for
passing messages. Two objects that share a channel may pass messages through that channel without
knowing each other's identities. Typical OO models and languages rely on object-based primitives for
direct method invocations, channel-based abstractions for IO and communication across wires, and
constructions such as event channels that may be viewed from either perspective.
Computation. Objects may perform four basic kinds of computation:
• Accept a message.
• Update internal state.
• Send a message.
• Create a new object.
This abstract characterization can be interpreted and refined in several ways. For example, one way to
implement a WaterTank object is to build a tiny special-purpose hardware device that only
maintains the indicated states, instructions, and connections. But since this is not a book on hardware
design, we'll ignore such options and restrict attention to software-based alternatives 