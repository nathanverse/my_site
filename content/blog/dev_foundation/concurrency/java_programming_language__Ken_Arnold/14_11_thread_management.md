+++
title = "7. Thread management"
date = "2025-01-08"
tags = [
    "markdown",
    "syntax",
]
+++

# Thread management

It is useful to group several threads into a unit, and if necessary, do:
+ Interrupt all threads in a group at once.
+ Placing a limit on the maximum priority of threads in a group.
+ Only access to certain files.

A thread group can be contained within another thread group, providing a hierarchy originating with system thread
group. A thread in a group can modify threads in that group, and threads in children groups. By modification, it
means invoking any method that could affect a thread's behavior such as changing its priority, or interrupting it.

Every thread belongs to a thread group (default thread group is one of the thread spawn it). You can specify 
the thread group for a new thread by passing it to the thread constructor.
```java
public Thread(ThreadGroup group, String name);
```

Security of threads tell you about level of permission a thread group is allowed to modify other threads. If 
you do otherwise, for example, set priority for a thread which is forbidden, `SecurityException` will throw.

> A thread can not change its thread group after the thread is created.

Several handful methods on `ThreadGroup`
```java
// The number of threads contained in the group
public int activeCount();

// Gets the thread groups contained in the group.
public int enumerate(Thread[] threadsInGroup, boolean recurse);
```

These methods above can be called statically on `Thread` instances.