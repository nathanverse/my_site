+++
title = "5. Cancellation and shutdown"
date = "2025-03-29"
tags = [
    "markdown",
    "syntax",
]
+++

# 1. Task cancellation
An activity is cancellable if external code can move it to completion before its normal completion. There are several reasons you want
to cancel an activity:
1. User clicked on the `cancel` button in a GUI app.
2. Search a problem space for a finite amount of time and choose the best solution.
3. App events: Decompose a problem into multiple tasks, once a task finds the solution, other tasks are interrupted.
4. ...


There is no safe way to preemptively stop a thread in Java, and therefore no safe way to preemptively stop a task. There are only cooperative
mechanisms, in which the task requested to be interrupted must agree do so.

Cancellation policy of a task involves, *how*, *when*, *what*:
+ How other code can request cancellation.
+ When the task checks whether cancellation has been requested.
+ What actions the task takes in response to a cancellation request.

Considering following codes:

{{< code_block id="1_prime_generator" language="java" title="PrimeGenerator">}}

@ThreadSafe
public class PrimeGenerator implements Runnable {
    @GuardedBy("this")
    private final List<BigInteger> primes
            = new ArrayList<BigInteger>();
    private  volatile boolean cancelled;
    
    public void run() {
        BigInteger p = BigInteger.ONE;
        while (!cancelled ) {
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        } 
    }
    
    public void cancel() { cancelled = true;  }
    
    public synchronized List<BigInteger> get() {
        return new ArrayList<BigInteger>(primes);
    } 
}
{{< /code_block >}}

{{< code_block id="1_prime_generator_user" language="java" title="Method to generate primes in one second">}}
List<BigInteger> aSecondOfPrimes() throws InterruptedException {
    PrimeGenerator generator = new PrimeGenerator();
    new Thread(generator).start();
    try {
        SECONDS.sleep(1);
    } finally {
        generator.cancel();
    }
    return generator.get();
}
{{< /code_block >}}

`PrimeGenerator` uses a simple cancellation policy: client code requests cancellation by calling cancel, `PrimeGenerator` 
checks for cancellation once per prime found and exits when it detects cancellation has been requested.

## 1.1. Interruption
Although [PrimeGenerator](#1_prime_generator) has the cancellation policy, it may encounter a severe problem that the code can never
check for cancellation if after the check it calls a long operation or blocking method. Imaging if the `primes.add(p)` takes long to run after
`PrimeGenerator` is canceled. In this case, we may want a shared state which indicates if the thread is already canceled, so that we can
check it in `primes.add(p)` and quickly exits the method.

A better way is taking advantage of thread methods, which supports interruption.
+ `Thread.interrupt()`: set the `interrupted` flag to true on the thread.
+ `Thread.isInterrupted()`: check if the `interrupted` true.
+ `Thread.interrupted()`: clear the `interrupted` flag and return the previous value.

Using these methods above, we can craft more responsive `PrimeGenerator` with two checking points, one in while flag, another in 
`queue.put()`:
```java
class PrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;
    
    PrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }
    
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted())
                queue.put(p = p.nextProbablePrime());
        } catch (InterruptedException consumed) {
            /*  Allow thread to exit  */
        }
    }
    
    public void cancel() { interrupt(); }
}
```

In fact, many blocking methods, such as `Thread.sleep` and `Object.wait` also check `interrupted` flag before it runs, enabling them to
return early if it detects the interrupt signal.

The well-behave task will not swallow the `interrupted` flag of the thread unless the task is taking control of what thread should behave
after the interruption. Only that do the caller methods can know if the  `interrupted` flag is set and respond accordingly.

## 1.2 Interruption policies.
Just as tasks should have a cancellation policy, threads should have an interruption policy, determining what should be done, and how 
quickly they react to interruption.

It is important to distinguish between how tasks and threads should react to interruption. 
A single interrupt request may have more than one desired recipient interrupting a worker thread in a thread pool can mean
both "cancel the current task" and "shut down the worker thread".

When writing cancellation code, rules are:
+ A task should not assume anything about the interruption policy of its executing thread unless it is explicitly designed to 
run within a service that has a specific interruption policy. 
+ You should know a thread's interruption policy before interrupting it.

## 1.3. Timed Run example
Above rules look complex, we will go through *Timed Run* example to understand these rules. *Timed Run* is the method that cancels
a given task after timeout expires like following:

```java
public static void timedRun(Runnable r, long timeout, TimeUnit unit);
```

Before implement the method, we should outline several requirements:
1. Calling this method should block the calling thread until the task is timeout or complete.
2. If the task meets an unchecked exception, `timedRun` method should throw that exception to the caller.

The first solution coming up may be to port the `Runnable` to another thread, using the `sleep()` method to wait, and `cancel` the
thread executing `Runnable` once it is timed out. Like that in [aSecondOfPrimes](#1_prime_generator_user).

However, this way we can't ensure both requirements:
1. For 1st, if the task completes earlier, the main thread still has to wait.
2. For 2nd, the folk thread will not leave any exception information to the main thread.

To solve problems mentioned, we can let the main thread execute the task, why creates a new thread to cancel the main thread if time out expires.

```java {linenos=table}
private static final ScheduledExecutorService cancelExec = ...;
public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
    final Thread taskThread = Thread.currentThread();
    cancelExec.schedule(new Runnable() {
        public void run() { taskThread.interrupt(); }
    }, timeout, unit);
    r.run();
}
```

The above solution seems appealing, until you realize the main thread may finish the task sooner and go to execute another task. After a while, the
thread scheduled when timing out cancel the main thread, and you won't know what may happen, probably crashing the server?. It is possible,
the code violates the first requirement also, if the `r.run()` when into long-running methods.

That is why you shouldn't cancel the thread when you don't know the cancellation policy of the thread.

We may then devise the more sophisticated solutions by combining both above solution ideas: 
+ The problem of the first solution was because we don't have any method to access to the unchecked exception of the
task in `Runable` , why don't we implement a new type of `Runnable` that can allow access to the exception through a method and ensure
that the main thread can exit throwing this exception. `join(timeout)` to the task thread, and access to that exception method 
may be a viable solution.
+ Using a dedicated thread in scheduler to cancel another thread should be kept, as we don't want to use `sleep()` method, which may block
the tasks that complete earlier.

Following is the code implementing the idea:
```java {linenos=table}
public static void timedRun(final Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
    class RethrowableTask implements Runnable {
        private volatile Throwable t;
        
        public void run() {
            try { r.run(); }
            catch (Throwable t) { this.t = t; }
        }
        
        void rethrow() {
            if (t != null)
                throw launderThrowable(t);
        } 
    }
    
    RethrowableTask task = new RethrowableTask();
    final Thread taskThread = new Thread(task);
    taskThread.start();
    cancelExec.schedule(new Runnable() {
        public void run() { taskThread.interrupt();  }
    }, timeout, unit);
    taskThread.join(unit.toMillis(timeout));
    task.rethrow();
}
```

The above code can solve the problems, but it looks complex. Moreover, using `join` like above doesn't allow us to determine whether
the `timedOut` return because the task is timeout or it completes earlier, either `taskThread.interrupt()` set the `interrupted` flag
first or the thread continues and exits first, which is risky to assumes.

## 1.4. Cancellation Via Future.
Now, let move on to use the new and convenient tools `Future`. Following is the code that address all the *Timed out task* problems
by using `Future`:

```java {linenos=table}
public static void timedRun(Runnable r,long timeout, TimeUnit unit) throws InterruptedException {
    Future<?> task = taskExec.submit(r);
    try {
        task.get(timeout, unit);
    } catch (TimeoutException e) {
        // task will be cancelled below
    } catch (ExecutionException e) {
        // exception thrown in task; rethrow
        throw launderThrowable(e.getCause());
    } finally {
        // Harmless if task already completed
        task.cancel(true);  // interrupt if running
    } 
}
```

1. **Line 3**: By default, we have a standard `Executor`, which is `taskExec` which we can use `submit()` method to assign a task to a specific 
thread to execute.
2. **Line 4**: The `submit()` method returns `Future` which the main thread can use `get(timeout, unit)` method to wait for the execution. This
method will throw `TimeoutException` if it times out (in this case the task may still be executing), unchecked exception of the task.
3. **Line 13**: The task can be canceled through `Future.cancel()` method. The boolean param, if `true` indicates that the thread and the task should be interrupted,
if `false`, it doesn't interrupt anything and only indicate that the task should not be executed if it hasn't been executed.

The `cancel()` is reasonable as we know what thread cancellation policy is as `Executor` is designed to abort the thread if the task is canceled.

## 1.5. Dealing with Non-interruptible blocking:
Many blocking library methods check the `interrupted` flag of the thread and throw `InterruptedException` if it is true. This allows
you easily craft the responsive method. However, there are some which haven't reacted to the interrupt signals. In this case, we may interrupt
thread by different methods, but this requires the greater awareness of such blocking methods and their consequences.

For example, synchronous socket I/O in java.io has `InputStream` and `OutputStream` with respectively `read()` and `write()` methods that aren't 
responsive to the interruption. However, close the underlying socket can make any threads blocked throw a `SocketException`.

Following is an example, in which `ReaderThread` which constantly read new data into `processBuffer` and is interrupted by using
`socket.close()`

```java {linenos=table}
public class ReaderThread extends Thread {
    private final Socket socket;
    private final InputStream in;

    public ReaderThread(Socket socket) throws IOException {
        this.socket = socket;
        this.in = socket.getInputStream();
    }

    public void interrupt() {
        try {
            socket.close();
        } catch (IOException ignored) {
        } finally {
            super.interrupt();
        }
    }

    public void run() {
        try {
            byte[] buf = new byte[BUFSZ];
            while (true) {
                int count = in.read(buf);
                if (count < 0)
                    break;
                else if (count > 0)
                    processBuffer(buf, count);
            } 
        } catch (IOException e) { /*  Allow thread to exit  */  }
    }
}
```

We can wrap the logic of canceling by closing the socket into a `Future` too by taking advantage of `newTaskFor` method of `Executor`.
`newTaskFor` is a factory method that allows us to create return own our `Future` after a `Callable` is submitted through
`Executor.submit`. We can implement the new type of `Future` which has `cancel` method to close the socket set to it. Following is the
implementation
```java {linenos=table}
public interface CancellableTask<T> extends Callable<T> {
    void cancel();
    RunnableFuture<T> newTask();
}

@ThreadSafe
public class CancellingExecutor extends ThreadPoolExecutor {
    ...
    protected<T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        if (callable instanceof CancellableTask)
            return ((CancellableTask<T>) callable).newTask();
        else
            return super.newTaskFor(callable);
    } 
}

public abstract class SocketUsingTask<T> implements CancellableTask<T> {
    @GuardedBy("this")
    private Socket socket;

    protected synchronized void setSocket(Socket s) {
        socket = s;
    }

    public synchronized void cancel() {
        try {
            if (socket != null)
                socket.close();
        } catch (IOException ignored) {
        }
    }

    public RunnableFuture<T> newTask() {
        return new FutureTask<T>(this) {
            public boolean cancel(boolean mayInterruptIfRunning) {
                try {
                    SocketUsingTask.this.cancel();
                } finally {
                    return super.cancel(mayInterruptIfRunning);
                }
            }
        };
    }
}
```