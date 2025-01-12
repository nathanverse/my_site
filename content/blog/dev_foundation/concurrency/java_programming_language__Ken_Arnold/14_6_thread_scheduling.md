+++
title = "3. Thread scheduling"
date = "2025-01-08"
tags = [
    "markdown",
    "syntax",
]
+++

Threads perform tasks, which have priority itself. Therefore, threads usually have a priority number associated to it. A thread is inactive when:
+ In call a block operation, `wait()`, `sleep()`
+ It is preempted, meaning the thread gives slot for another thread to run (a thread with higher priority, or because the current CPU cycles the current
thread allowed to use is exceeded)

> **⚠️ Warning:**
> Use priority only to affect scheduling policy for efficiency. Do not rely on thread priority for algorithm correctness. Attempting to write
> not thread-safe code by assuming the current thread can execute exclusively with higher priority, while there are other lower-prioritized threads
> access to that code, would potentially fail.

When coding a multithreaded application, consider following:
1. Set each thread with a proper priority with `setPriority` on `Thread` object, for example, 
threads that respond to user input often has higher priority.
2. Normally, values for the thread priority of your app should be `NORM_PRIORITY` with a small delta around it. Be cautious to use constant
`MIN_PRIORITY` or `MAX_PRIORITY`, because on some systems, this might affect other important programs.

### Voluntary scheduling`
You can relinquish the current thread's use of the CPU, by calling following `Thread` static methods:
1. `sleep(long millis)`: Sleep for millis of time.
2. `yield()`: signal the system that, the current thread need not run at the present time, give the chance for another thread to run.
This call might be ignored by the system. Use this when you think it is better for other threads to run in your application, instead of this thread.