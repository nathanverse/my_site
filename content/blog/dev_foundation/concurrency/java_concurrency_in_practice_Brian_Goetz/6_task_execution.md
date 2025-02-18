+++
title = "4. Task execution"
date = "2025-02-17"
tags = [
    "markdown",
    "syntax",
]
+++


# 1. Executing tasks in threads
We need to identify boundary for each task to foster task independence, which is coupled with task execution policy can exhibit:
+ Better concurrency as independent tasks can be executed in parallel if there are adequate processing resources.
+ Good throughput and responsiveness.
+ Graceful degradation.

For example, oftentimes, we see server application choose to separate each client request as a task boundary. This helps a task not being
affected by other tasks. Also, one message is easy to digest and require a very small percentage of server's total capacity.


## 1.1. Executing tasks sequentially
Following os a simple web server, which execute tasks sequentially. This server has poor throughput and responsiveness, as a request comes in
, it might be blocked by I/O operation, leading to unnecessary delay for other incoming requests.

```java {linenos=table}
class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            Socket connection = socket.accept();
            handleRequest(connection);
        }
    } 
}
```

## 1.2. Explicitly creating threads for tasks
We attempt to create a thread for each request coming in following code. This definitely offers higher throughput and responsiveness.

```java {linenos=table}
class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final  Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                } };
            new Thread(task).start();
        } 
    }
}
```

## 1.3. Disadvantages of unbounded thread creation. {#unbounded-thread}
However, the thread-per-task approach has some practical drawbacks:
1. Thread lifecycle overhead: Thread creation and teardown are not free, it introduces significant latency if the requests are frequent and
lightweight.
2. Resource consumption: Key ideas is to add as many threads sufficiently to keep CPUs busy. Once they are, adding new threads will just
introduce performance cost due to contention between threads. Memory usage will increase as each thread being added as well, 
and might place a pressure on the garbage collector, which further makes things worse.
3. Stability: there are limit on the number of threads that are allowed to be created for an application, hitting this number can result in
`OutOfMemorryError` which is risky to recover. It is far easier to structure your program to avoid hitting this limit.
4. Possibility of server crash: This method doesn't provide any means to place the limit of threads can be created. Malicious user, or enough
ordinary users, can attempt to push the traffic to exceed your app maximum capacity. An app always need to provide high availability and grateful degradation
under load.

# 2. The executor framework
To address the problems of sequential and thread-per-task approach, we can use the implementation of the `Executor` to limit the size 
of threads can be created - thread pool.

It is worth to mention what is `Executor`. Following is its interface:

```java {linenos=table}
public interface Executor {
    void execute(Runnable command);
}
```

`Executor` represents executor policy of an application (or part of it) which specifies "what, where, when, and how" of task execution, including:
+ In what thread will tasks be executed? 
+ In what order should tasks be executed (FIFO, LIFO, priority order)? 
+ How many tasks may execute concurrently? 
+ How many tasks may be queued pending execution? 
+ If a task has to be rejected because the system is overloaded, which task should be selected as the victim, and
how should the application be notified? 
+ What actions should be taken before or after executing a task?

> ✍️ Excerpt:
> Separating the specification of execution policy from task submission makes it practical to select an execution policy at 
> deployment time that is matched to the available hardware.

For example, you can benchmark to check the number of threads sufficient to run on your limited memory and CPUs and tool the `Executor` 
accordingly.

Following is a webserver using `Executor`:
```java {linenos=table}
class TaskExecutionWebServer {
    private static final int NTHREADS = 100;
    private static final Executor exec
            = Executors.newFixedThreadPool(NTHREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
        }
        exec.execute(task);
    }
}
```

As you can see, the web sever uses `Executor` as its policy to execute concurrent requests. It simplifies the responsibility of the web server
by only submitting the request to the `Executor` , which will be responsible to execute the tasks by its internal policy instead of spreading this
implementation in the web server itself.

You can implement execution policy like *thread-per-task* web server as well as sequential one:
```java {linenos=table}
public class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    }; 
}
```

`Executor` can also opens the door to all sorts of additional opportunities for tuning, management, monitoring, logging, error reporting, 
and other possibilities that would have been far more difficult to add without a task execution framework.


## 2.1. Thread pools:
Thread pools executor place a limit on number of worker threads can be created. Each worker thread, in turn of execution, takes a task
from a task queue (which can be bounded or unbounded, though unbounded one can cause out of memory) and execute.

*Thread-pool* approach not only addresses those problems mentioned in [Section 1.3]({{< ref "#unbounded-thread" >}}), as it creates thread
only there are number of requests exceeding the current size of the pool, it limits on the number of times creating the threads, and thus
improving responsiveness.

There are several implementations of thread pool, which you can set up using static factory methods in `Executor`:
+ `newFixedThreadPool`: A fixed‐size thread pool creates threads as tasks are submitted, up to the maximum pool size, 
and then attempts to keep the pool size constant.
+ `newCachedThreadPool`: A cached thread pool has more flexibility to reap idle threads when the current size of the pool 
exceeds the demand for processing, and to add new threads when demand increases, but places no bounds on the size of the pool.
+ `newScheduledThreadPool`. A fixed‐size thread pool that supports delayed and periodic task execution, similar to `Timer`.
+ `newSingleThreadExecutor`. A single‐threaded executor creates a single worker thread to process tasks, replacing it if it dies 
+ unexpectedly. Tasks are guaranteed to be processed sequentially according to the order imposed by the task queue 
+ (FIFO, LIFO, priority order).

The `newFixedThreadPool` and `newCachedThreadPool` factories return instances of the general‐purpose `ThreadPoolExecutor`, 
which can also be used directly to construct more specialized executors. 

## 2.2. Executor lifecycle
Because executor is a service that run asynchronously to the app, you need a mechanism to shut down it gratefully, or else, 
the JVM will not exist as there are still thread running.

`ExecutorService` extends `Executor` to include several methods to manage the lifecycle of the `Executor` 

```java {linenos=table}
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    //  ... additional convenience methods for task submission
}
```

+ `shutdown`: shut down the executor gratefully, new tasks will not be accepted. However, tasks are accepted but still are not run 
still are allowed to be run.
+ `shutdownNow`: shutdown abruptly, tasks are queued but are not started yet will not begin.
+ `awaitTermination`: after shutting down the service, you can await on this method to wait until the executor stop running.

## 2.3. Delayed and periodic tasks.
Delayed tasks are tasks that we ask to execute after a certain period has passed, while periodic tasks are tasks that are executed
at a specific time overtime.

`Timer` is an obsolete use for this kind of problem. Its drawbacks are:
+ Using only one thread: another `TimerTask` might be not executed at the time it
is supposed to due to waiting other threads.
+ Handle exception poorly: if a task throw exception while running, it cancels the whole `Timer`. In this case, we usually need it
to recover and run other tasks instead.
  + Also, if the initial task throw exception to fast, it causes the main thread to cancel too. As illustrated in following code, the code
  is supposed to end after 6 seconds, but it might return after 1 second.


```java {linenos=table}
public class OutOfTime {
    public static void main(String[] args) throws Exception {
        Timer timer = new Timer();
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(1);
        timer.schedule(new ThrowTask(), 1);
        SECONDS.sleep(5);
    }
    
    static class ThrowTask extends TimerTask {
        public void run() { throw new RuntimeException(); }
    } 
}
```

Using `ScheduledThreadPoolExecutor` you can address these problems. Two implementations you might want to test include `DelayQueue` and
`BlockingQueue`

## 2.3. Finding the exploitable parallelism

