+++
title = "Thread terminnation"
date = "2025-01-07"
tags = [
    "markdown",
    "syntax",
]
+++

# Terminate threads

A thread terminates when:
+ The run method returns normally.
+ The run method completes abruptly.
+ The application terminates.

When a thread is terminated, all locks must be released, otherwise it would make other threads to wait forever.

There is some case you would want a thread to take control the CPU for while, and then cancel it if you want.
For example, imagine a user `Cancel` button is triggered while another thread is in a transaction,
making request to the database. You'd prefer the thread would be canceled after completely returning from the
transaction. In that case, following pattern is popular.

```java
// Cancel button action
thread2.interrupt();
```

```java
// The thread in a transaction
while(!interrupted()){
    // do the transaction
}
```

You should check constantly for interruption, and avoid a nested long-running code if you want to interrupt immediately
once receiving interrupt signal.

Once thread is interrupted, the method, or class should involve clean up operation. For example:
```java
void doCleanUp(){
    // Possible return exception after the interruption
    this.file.close();
}

void tick(int count, long pauseTime) {
    try {for (int i = 0; i < count; i++) { System.out.println('.');
        System.out.flush();
        Thread.sleep(pauseTime);
    }
    } catch (InterruptedException e) {
        // Clean the clean up flag
        Thread.interrupted();
        
        // Do clean up
        doCleanUp();
        
        // Some other cleanups which might be aborted.
        otherCleanUp();
        
        Thread.currentThread().interrupt();
    }
}

```

The above method calls `Thread.sleep(pauseTime)`, which might throw `InterruptedException` if the thread is interrupted. If it happened, we catch it, clear the thread state for doing cleanup. This is necessary because the cleanUp process might involve
operations that could throw an exception too, which might leave the current operation in the unclean state.


> Always clean state with `Thread.interrupted()` before doing cleanup operation.


> *** ⚠️ Warning***
> You should not clear interrupted flag, or catch exception and then let it run normally. Otherwise, your code would ignore every interrupted signal from system, and leave the code malicious.

## Waiting for a thread to complete
You could use `join` on the target thread to make the current thread wait until it terminates. Internally, it is equal to:

```java
while (isAlive())
    wait();
```

# Terminate the application
Apps have two kinds of threads:
+ Daemon thread.
+ User thread.

Your app starts with a user thread, and spawn new threads as daemon or user threads, if it leave as default, either the thread is daemon or not will inherit from the parent thread. 

The application will be terminated once all user threads are terminated. You might either use a special method `exit` from `Runtime` or `System` object to terminate the application too.

Many library classes create a new user thread within its implementation. Therefore, if you want to terminate the app in this case, termination of `Runtime` and `System` object is necessary.