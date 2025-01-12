+++
title = "8. Threads and Exceptions"
date = "2025-01-12"
tags = [
    "markdown",
    "syntax",
]
+++

# Threads and exceptions

A thread will be terminated if it meets an uncaught exception. The uncaught exception is usually the sign of a serious error, their
occurrence needs to be tracked in some way. To enable this, every thread can have associated with it an instance of `UncaughtExceptionHandler`
which has a method:
```java
public void uncaughtException(Thread thr, Throwable exc);
```

that can handle the exception, and the thread throws exception.

When the thread meets an uncaught exception. Following will happen.
1. Get `UncaughtExceptionHandler` handler of the thread, which can be set through `setUncaughtExceptionHandler` and call `uncaughtException()`/
2. If the handler is not set, the `ThreadGroup` of the thread is returned, in this case
   1. `uncaughtException()` will be called on the handler of that `ThreadGroup` or its father groups.
   2. In case no parent `ThreadGroup` has the handler, handler will be taken from the static method `Thread.getdefaultUncaughtExceptionHandler`
   3. In case `Thread.getdefaultUncaughtExceptionHandler` returns nothing, the stack trace will of the exception will be printed.
3. If thread is terminated before `getUncaughtExceptionHandler` is called, the method will return `null`.

Therefore, you can control the handling of uncaught exceptions, by:
+ At the system level, by setting a default handler using the static Thread class method setDefaultUncaughtExceptionHandler. This operation is security checked to ensure the application has permission to control this. 
+ At the group level, by extending ThreadGroup and overriding it's definition of uncaughtException. 
+ At the thread level, by invoking setUncaughtExceptionHandler on that thread.

Therefore, if there is something that make your code not print uncaught exceptions of the thread, look at its handler.

## Don't call `stop()`

Like `interrupt`, `Thread.stop()` is used for terminate thread, there are several differences:
1. The thread throws `ThreadDepth` exception, unlike `interrupt` which assign a flag on the current thread.
2. The terminated thread will release the lock immediately when it is executing the code in synchronized block.

This causes several issues:
1. If methods on the upper stack catch the exception and then not rethrow, the code will be malicious.
2. As it aborts immediately in the synchronized block, the code can corrupt when just partially complete the current operation.

So don't use `stop()`.