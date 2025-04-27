+++
title = "9. GUI applications"
date = "2025-04-23"
tags = [
    "markdown",
    "syntax",
]
+++

# 1. Why are GUIs single-threaded?
There have been many attempts to write multithreaded GUI frameworks, but because of the persistent problems
with race conditions and deadlock, almost all GUI frameworks transition to use single-threaded approach. For example, 
AWT originally tried to support greater degree of multithreaded access, and the decision to make Swing single-threaded was based largely on experience with  AWT

Concurrency hazards often occur in GUI framework, because the program often deals with many events that modify
different states of components. These components can depend on each other, making it easy to encounter deadlock.

The MVC pattern also one of the causes of deadlock. The controller can call into view, and model, while view can 
query model states by callbacks. The pattern may potentially lead to inconsistent lock ordering.

Single-threaded approach achieves thread safety via thread confinement. It is the developer responsibility
to make sure visual components and data models are properly confined.

## 1.1. Sequential event processing.
Events such as mouse clicks, key presses, or timer expiration are handled sequentially. If one task is lengthy,
it could make user inputs, like press button, or visual feedback appear to be frozen. Therefore, all 
I/O blocking, or CPU intensive tasks must be handed off to other threads.

## 1.2. Thread confinement in Swing.
All data models and components in Swing are only allowed to be created and updated by the UI thread.

Sending tasks to be executed on the UI thread or subscribe listeners to events from other threads 
is through `SpringUtilities` methods, which ensure to be thread-safe:
1. `SwingUtilities.isEventDispatchThread`: which determines whether the current thread is the event thread;
2. `SwingUtilities.invokeLater`: which schedules a Runnable for execution on the event thread (callable from any thread);
3. `SwingUtilities.invokeAndWait`: which schedules a Runnable task for execution on the event thread and blocks the 
current thread until it completes (callable only from a non‐GUI thread);
4. methods to enqueue a repaint or revalidation request on the event queue (callable from any thread); and
5. methods for adding and removing listeners (can be called from any thread, but listeners will always be invoked
   in the event thread).

The method exposed in 1,2,3 looks the same as a single-threaded executor. However, as Swing predate java executors, it is not
based on them. However, you can easy to implement one by using a single-threaded executor.

# 2. Short running GUI tasks