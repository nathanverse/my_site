+++
title = "wait, notifyAll, notify"
date = "2025-01-07"
tags = [
    "markdown",
    "syntax",
]
+++

# wait, notifyAll, notify
In some circumstances, you might want a thread to notify another thread to do its work when some condition occurs. Example can be, you
have a `Print` class, where people can add `PrintJob`. You would to model thread like
1. A thread to receive job.
2. A thread to do the job.

This model is efficient, because clients could receive a return immediately, and wait until their jobs get done.

In this model, the thread performs a job might have to wait if the job queue is empty. Using while loop in this case might cost CPU resource
a lot (busy-waiting). Instead, you could do something like:


```java
class PrintQueue {
    private SingleLinkQueue<PrintJob> queue =
        new SingleLinkQueue<PrintJob>();
    public synchronized void add(PrintJob j) {
        queue.add(j);
        notifyAll();     // Tell waiters: print job added
    }
    public synchronized PrintJob remove() throws InterruptedException {
        while (queue.size() == 0)
            wait();      // Wait for a print job
        return queue.remove();
    }
}
```

+ `notifyAll()`: Notify to awake all threads that are waiting on the object, in this case `PrintQueue`.
+ `wait()`: Release the object lock that is being acquired, and wait until being notified.
+ `notify()`: different to `notifyAll`, this method chooses a random thread to notify.


With these tools, the thread responsible for doing the job would become inactive and doesn't use any CPU resource.

Generalize the problem, typically, you would see the pattern of thread communication like this:
```java
synchronized void doWhenCondition() { 
    while (!condition)
        wait();
    // ... Do what must be done when the condition is true ...
}

synchronized void randomFunction(PrintJob j) {
    // Do something
    condition = true;
    notifyAll();
}
```


Using `notifyAll` or `notify` would be based on your implementation requirement. You `notify` when:
+ All threads are waiting for the same condition 
+ At most one thread can benefit from the condition being met 
+ This is contractually true for all possible subclasses

> **⚠️ Warning:**
> It is also possible that some virtual machine implementations will allow so-called "spurious wakeups"
> to occur when a thread returns from wait without being the recipient of a notification, interruption
> This is one of the reason that `wait` should always be performed in a loop that tests the condition 
> being waited on.