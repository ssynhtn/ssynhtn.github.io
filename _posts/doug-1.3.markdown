1.3 Design Forces
This section surveys design concerns that arise in concurrent software development, but play at best
minor roles in sequential programming. Most presentations of constructions and design patterns later
in this book include descriptions of how they resolve applicable forces discussed here (as well as
others that are less directly tied to concurrency, such as accuracy, testability, and so on).
并发编程中浮现了一些在顺序编程中很少会考虑到的问题:
安全性，
线程的活跃性，
代码运行的性能，
代码的可重用性

One can take two complementary views of any OO system, object-centric and activity-centric:
Under an object-centric view, a system is a collection of interconnected objects. But it is a structured
collection, not a random object soup. Objects cluster together in groups, for example the group of
objects comprising a ParticleApplet, thus forming larger components and subsystems.
Under an activity-centric view, a system is a collection of possibly concurrent activities. At the most
fine-grained level, these are just individual message sends (normally, method invocations). They in
turn organize themselves into sets of call-chains, event sequences, tasks, sessions, transactions, and
threads. One logical activity (such as running the ParticleApplet) may involve many threads.
At a higher level, some of these activities represent system-wide use cases.
Neither view alone provides a complete picture of a system, since a given object may be involved in
multiple activities, and conversely a given activity may span multiple objects. However, these two
views give rise to two complementary sets of correctness concerns, one object-centric and the other
activity-centric:
Safety. Nothing bad ever happens to an object.
Liveness. Something eventually happens within an activity.