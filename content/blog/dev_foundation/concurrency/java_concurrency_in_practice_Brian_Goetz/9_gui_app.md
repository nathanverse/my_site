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

# 2. Long running GUI tasks.
Listener of UI events can evoke long-running tasks such as spell checking, background compilation, or fetching remote resources. To avoid
freezing the app, we need to offload them to background threads.

Offloading long-running tasks to elastic `Executor` can be a great choice. Only rarely do GUI applications initiate a large number of
long-running tasks. Following is an example of how we are going to offload:

```java
ExecutorService backgroundExec = Executors.newCachedThreadPool();
...
button.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        backgroundExec.execute(new Runnable() {
            public void run() { doBigComputation(); }
        });
}});
```
We may need to have some sort of visual feedback indicating when the task completes. For example, we may need to set the label to `busy`
before trigger the task and set it back to `idle` when it is done.

```java {linenos=table}
button.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        button.setEnabled(false);
        label.setText("busy");
        backgroundExec.execute(new Runnable() {
            public void run() {
                try {
                    doBigComputation();
                } finally {
                    GuiExecutor.instance().execute(new Runnable() {
                        public void run() {
                            button.setEnabled(true);
                            label.setText("idle");
                        }
                    });
                }
            }
        });
    }
});
```

The code starts getting complicated with 3 layers. This sort of "thread hopping" is typical of handling long‐running tasks in GUI applications.

## 2.1. Cancellation
The task may take long so we need to provide the button where users can cancel when they want. `Future` comes in handy to handle this.
```java {linenos=table}
Future<?>  runningTask = null;    // thread-confined
...
startButton.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent e) {
        if (runningTask != null) {
           runningTask = backgroundExec.submit(new Runnable() {
              public void run() {
                 while (moreWork()) {
                    if (Thread.currentThread().isInterrupted()) {
                       cleanUpPartialWork();
                       break;
                    }
                    doSomeWork();
                 }
              }
           });
        } 
    });
});

cancelButton.addActionListener(new ActionListener() {
    public void actionPerformed(ActionEvent event) {
        if (runningTask != null)
            runningTask.cancel(true);
    }
});
```

## 2.2. Progress and Completion indication
Upon task completion, we often need to update the UI, for instance, by hiding the cancel button. Manually handling this, 
perhaps in a cleanup method, can lead to poorly structured code where task logic and UI updates are awkwardly mixed. 
Furthermore, reporting progress from the running task presents similar challenges, potentially complicating the code further.

A common pattern emerges: the core task logic needs to run on a background thread, while completion notifications and progress 
updates must safely interact with the UI thread. This suggests creating an abstract task class. Such a class would require
developers to implement abstract methods for the background computation, the completion logic, and progress reporting. 
The base class itself would internally manage the necessary thread switching, significantly simplifying the developer's
code by separating concerns.

Following is the implementation of the abstract task class:
```java {linenos=table}
abstract class BackgroundTask<V> implements Runnable, Future<V> {
    private final FutureTask<V> computation = new Computation();
    private class Computation extends FutureTask<V> {
        public Computation() {
            super(new Callable<V>() {
                public V call() throws Exception {
                    return BackgroundTask.this.compute() ;
                }
            }); 
        }
        
        protected final void done() {
            GuiExecutor.instance().execute(new Runnable() {
                public void run() {
                    V value = null;
                    Throwable thrown = null;
                    boolean cancelled = false;
                    try {
                      value = get();
                    } catch (ExecutionException e) {
                      thrown = e.getCause();
                    } catch (CancellationException e) {
                      cancelled = true;
                    } catch (InterruptedException consumed) {
                    } finally {
                      onCompletion(value, thrown, cancelled);
                    }
                } 
            }
        }; 
    });
            
    protected void setProgress(final int current, final int max) {
        GuiExecutor.instance().execute(new Runnable() {
            public void run() { onProgress(current, max); }
        });
    }
    // Called in the background thread
    protected abstract V compute()  throws Exception;
    // Called in the event thread
    protected void onCompletion(V result, Throwable exception,
                                     boolean cancelled)  { }
    protected void  onProgress(int current, int max)  { }
    // Other Future methods forwarded to computation
}
```

Implement your task's core logic within the `compute()` method. For completion notifications, use `onCompletion()`, and for 
indicating progress, use `onProgress()`.

When this task is scheduled on an Executor, the `compute()` method is the first to be invoked. The `FutureTask` class conveniently 
calls its `done()` method upon the task's completion. This `done()` method executes on the UI thread, guaranteeing that `onCompletion()`
also runs on the UI thread. Similarly, when you update task progress within `compute()` using `setProgress()`, the `onProgress()` 
method is also executed on the UI thread.

# 3. Shared data models
## 3.1. Thread-safe data models
In the GUI app, it is best to comply to the thread confinement principle. However, in case you need to share model data between
background and event threads, you can also do so if you make sure that the model is thread safe.
## 3.2. Split data models
As mentioned, the UI views may need to be updated by a shared model used by both event thread and background thread. In this approach,
ensuring thread-safe often requires the event thread to retrieve the snapshot of the model, which may be inefficient if the data model
is large of may be frequently updated.

Another approach is to send incremental changes. Whenever the shared model changes states, it sends only that change to update the view.
This helps save memory as we only need to send necessary changes to the presentation models.

In summary, in whatever approach, you need to ensure:
1. Thread-safe models.
2. The event thread remains responsive.
3. Balance between model availability and write performance.