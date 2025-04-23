+++
title = "8. Thread pools"
date = "2025-04-11"
tags = [
    "markdown",
    "syntax",
]
+++

# 1. Implicit couplings between tasks and execution policies.

While the `Executors` offers the substantial flexibility in specifying and modifying execution policies. Not all tasks are compatible
with all execution policies, these tasks include:

+ Dependent tasks: Tasks that depend on the timing, results, or side effects of other tasks. When you submit such tasks, you implicitly
create constraints on the execution policy that must be carefully managed to avoid thread starvation problem where the thread may wait
a condition that never is fulfilled. (more on this is discussed in 1.1).
+ Tasks that exploit thread confinement: Tasks use thread confinement technique may suit with single-threaded executor rather than
multithreaded one.
+ Response-time sensitive tasks: It is troublesome for letting user wait the response after clicking button. Putting such tasks into 
executor with full of long-running tasks often are not suitable.
+ Tasks that use `ThreadLocal`: states in `ThreadLocal` can be shared between thread. If you want to submit that task in the executor,
you must ensure the thread has the life span longer the given tasks. Be aware because most executors may interrupt a current thread and
use another thread to serve your task.

These reasons explain that thread pools work best when tasks are homogeneous and independent.

## 1.1. Thread starvation deadlock
Tasks depend on another task in the queue (like waiting their results or side effect) may cause deadlock on the thread. For example,
a task that puts other tasks in the queue of single-threaded executor and wait for their results always faces deadlock.

This is the same with multithreaded executor when a task may wait another task that is never picked up, because other threads are too busy.
There are various reasons that cause deadlock too. The idea is whenever your queue contains dependent tasks, you are risking with thread 
starvation. In these cases, you may want to document down the configurations of the executor, like minimal pool sizing to avoid the issue.

Other resource constraints, such as JDBC connections, should be aware as well. Such resource can block your thread from execution if it reaches
the limit. In this case, 10 connections mean only 10 threads can be executed concurrently.

## 1.2. Long-running tasks
One solution to address the issue where long-running tasks clog the queue, preventing other smalls tasks from making progress, is use timed
resource waits instead of unbounded wait, like putting the timeout on `Thread.join`, `BlockingQueue.put`. When a task is timed out, the
task could be aborted or queued again later, allowing other tasks to make progress.

# 2. Sizing thread pools
To size the thread pool properly, you need to understand your computing environment, your resource budget, and nature of your tasks:
1. How many processors does the deployment system have? How much memory?
2. Do tasks perform mostly computation, I/O, or some combination?
3. Do they require the scare resource, such as JDBC connection?

For compute-intensive tasks, the suitable number of threads typically is the number of processor plus 1. Adding one more thread meant to cover
cases where tasks are interrupted. Even if it is CPU-bound task, there is case it is temporarily stopped due to page fault, preemptive
multitasking, or synchronization.

We often want a larger pool for tasks that include I/O or other blocking operations (it is not the case if you are using non-blocking
technique) because most threads will not be schedulable at all times.

In another word, you need to estimate the ratio of waiting time to compute time for your tasks; this estimate need not be precise and can be 
obtained through pro‐filing or instrumentation. Alternatively, you can benchmark different pool sizes under heavy loads and observe the
level of CPU utilization to choose the appropriate size.

Given these definitions, following is the formula:

[img](/pool_size_formula.png)

As mentioned other resources also a factor determining your pool size, like JDBC connections. Although, these are easier to estimate the correct
pool size, just get the total of resources divided by the number of resources need per task.

# 3. Configuring `ThreadPoolExecutor`

Static methods such as `newCachedThreadPool`, `newFixedThreadPool` in `Executor` class provides `ThreadPoolExecutor` as a basic implementation
for `ExecutorService`. If you don't satisfy with these default configurations, you can customize through `ThreadPoolExecutor`
constructors.

## 3.1. Thread creation and teardown.
Three things you may want to configure are:
+ Core pool size: which is the size the executor will try to maintain throughout its lifecycle even if there is no task currently being
executed.
+ Maximum pool size: Maximum threads will be created if the queue is fully filled with tasks.
+ Keep-alive time: Threads with life span exceeding this time will be marked as idle and can be terminated.

In `newFixedThreadPool`, the core pool size and the maximum pool size are equal, meaning the number of threads remains constant. 
Conversely, `newCachedThreadPool` has a core pool size of `0` and a maximum pool size of `Integer.MAX_VALUE`, effectively 
creating an infinitely expandable thread pool that shrinks automatically when threads become idle.

## 3.2 Managing Queued tasks
If the clients throw requests at the servers faster than the servers can handle them, queuing tasks may lead to memory exhaustion, and you
may want to use some techniques to throttle arrival rate (like rate limiter). Even before you run out of memory, response time will get 
progressively worse as the task queue grows.

`ThreadPoolExecutor` allows you to supply a `BlockingQueue` to hold tasks awaiting execution. There are three basic approaches 
to task queuing:
+ Unbounded queue: the drawback of this type of queue was mentioned.
+ Bounded queue: You have to determine what to do with incoming tasks when the queue is full (this called saturation policies, discussed in 3.3)
+ Synchronous handoff: Handing off tasks directly to thread. In this method, there has to be a thread to accept the handoff before the
main thread moves on. If there is no thread is waiting, and the maximum count hasn't been reached, the executor will create a new thread
to accept the handoff. Otherwise, the task will be rejected by the saturation policy. This method can be more efficient than traditional queueing method
, especially for large pool size,, because there is no delay in putting and fetching task. `newCachedThreadPool` factory uses this kind of 
queue.

You can also use `PriorityBlockingQueue`, if you want to execute certain tasks first before the others.

As you already known, if your tasks depend on each other, unbounded queue and thread pool is suitable than bounded ones because they
prevent thread starvation. In this case, you may want to use `newCachedThreadPool` factory.

## 3.3 Saturation policies
There are 4 common saturation policies:
- `AbortPolicy`: throw an exception `Rejected-ExecutionException` if the queue filled up
- `DiscardPolicy`: Silently discard the requested task.
- `DiscardOldestPolicy`: Discard the old task, try to insert the newly added tasks. (usually is not preferred in `PriorityBlockingQueue`).
- `CallerRunsPolicy`: When the queue is full, the task will be executed in the caller thread (which calls `execute()` on `Executor`), this
would lead to the point where the main thread get blocked and stop receiving connections through `accept()`. The requests will be then
in waiting queue under TCP layer (which has finite queue) until the server can't handle anymore and stop the further requests. This
policy is usually favored.

Though there is no built-in policy to block the caller threads, you can use `Semaphore` to implement it, following is the example code:

```java
@ThreadSafe
public class BoundedExecutor {
    private final Executor exec;
    private final Semaphore semaphore;
    public BoundedExecutor(Executor exec, int bound) {
        this.exec = exec;
        this.semaphore = new Semaphore(bound);
}
    public void submitTask(final Runnable command) throws InterruptedException {
        semaphore.acquire();
        try {
            exec.execute(new Runnable() {
                public void run() {
                    try {
                        command.run();
                    } finally {
                        semaphore.release();
                    } 
                }
            });
        } catch (RejectedExecutionException e) {
            semaphore.release();
        }
    } 
}
```

## 4. Extending `ThreadPoolExecutor`

`ThreadPoolExecutor` is designed for extension, providing several hooks like:
+ `beforeExecute`: Executed in the same thread with the task before executing the task. Tasks throw in this hook will not be executed.
+ `afterExecute`: Executed after the task is done or throw.
+ `terminate`: Execute after the executor is executed.

You can add logging like the elapsed time a task is executed, the number of tasks have been executed in the executor, notification, or some statistics in these hooks.

## 5. Parallelizing Recursive algorithm
Loops whose bodies contain nontrivial computation or perform potentially blocking I/O are good candidates for parallelization.

One good application for parallelization is the Puzzle problem, which has the current position, and the goal position needed to achieve to win the game. It often has:
1. As being said, the initial position and the target position.
2. The set of legal moves from a position to another position.
3. A way to move to the next position.
4. way to check whether the current position is the target position.

Defined as following:

```java
public interface Puzzle<P, M> {
    P initialPosition();
    boolean isGoal(P position);
    Set<M> legalMoves(P position);
    P move(P position, M move);
}
```

Navigating problems like this often involves exploring all possible states or positions to pinpoint the desired outcome. Algorithms such as Depth-First Search (DFS) or Breadth-First Search
(BFS) are common approaches, frequently implemented using recursion. To boost performance, we can introduce concurrency by distributing the task of examining each state across a 
pool of threads.

Consider implementing a Minesweeper puzzle as a practical exercise. The challenge involves:

+ **Initial state**: A grid where all squares are initially unrevealed.
+ **Goal state**: Revealing all non-mine squares on the board.

Key implementation considerations include:
+ **Early termination**: Once a thread identifies the goal state, it should signal all other threads to halt and return the result.
+ **Scalability and responsiveness**: Given potentially large boards, your program should allow the user to interrupt the search and exit gracefully, 
ensuring no threads remain active or 'leak'.