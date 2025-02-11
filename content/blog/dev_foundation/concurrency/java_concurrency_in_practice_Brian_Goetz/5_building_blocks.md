+++
title = "3. Building blocks"
date = "2025-02-02"
tags = [
    "markdown",
    "syntax",
]
+++

The Java platform libraries include a rich set of concurrent building blocks
, like:
+ Thread-safe collections
+ Variety of synchronizers, being able to coordinate the control flow
of cooperating threads.

This chapter will get you covered about those concurrent building blocks.


# 1. Synchronized collections.
Java libraries support synchronized collection classes such as
`Vector` and `Hashtable`, part of the original JDK, 
as well as their cousins added in JDK 1.2, 
the synchronized wrapper classes created by the `Collections.synchronizedXxx` 
factory methods.

## 1.1. Problems with synchronized collections.
Considering following static methods, for the `Vector` class.

```java {linenos=table}
public static Object getLast(Vector list) {
    int lastIndex = list.size() - 1;
    return list.get(lastIndex);
}

public static void deleteLast(Vector list) {
    int lastIndex = list.size() - 1;
    list.remove(lastIndex);
}
```

If we have two threads, each doing operation of these methods concurrently. Like
following images

![img](/vector_problem.png)

The thread calling `getClass` might throw `ArrayIndexOutOfBoundsException`,
as the `size` it obtained from the `vector` is outdated.

The same occurs for the following code, which assumes `vector.size()` will
not be changed.
```java {linenos=table}
for (int i = 0; i < vector.size(); i++)
    doSomething(vector.get(i));
```

This occurs because we have to maintain the correct invariants between
the `size` of the vector and the underlying array of `vector` itself. To
address the problem, we can resort to `client-side` lock. For
example

{{< code_block id="1_1_vector_iterator_solve" language="java" title="Solving vector synchronized problems">}}
synchronized (vector) {
  for (int i = 0; i < vector.size(); i++)
  doSomething(vector.get(i));
}
{{< /code_block >}}

However, the solution above might cause the app to suffer from 
performance if `doSomething` is a lengthy operation, or the size of
the collection is large.

An alternative to locking the collection during iteration is to clone the
collection and iterate the copy instead. Cloning the collection has an obvious 
performance cost; whether this is a favorable tradeoff depends on many factors 
including the size of the collection, how much work is done for each element, 
the relative frequency of iteration compared to other collection operations, 
and responsiveness and throughput requirements.


## 1.2. Iterators and `ConcurrentModificationException`
The `Iterator`, which is used to iterate a `Collection` can fail-fast
meaning that if they detect that the collection 
has changed since iteration began, they throw 
the unchecked `ConcurrentModificationException`

But the implementation of `Iterator` is "good-faith effort", `hasNext()`
 and `next()` will check for a modification count which is associated with the 
collection to determine if they throw the exception. However, the 
modification count check is done without synchronization,
so there is a risk of seeing a stale value of the modification count 
and therefore that the iterator does not realize 
a modification has been made.

## 1.3 Hidden iterators
There might be certain cases the code implicit call iteration on the list,
like if you call `print` with a collection. Each element of the collection will
be iterated and the `toString()` method is called on it recursively. The
author of the code often forget and don't use any synchronization for
this hidden iteration. For example

```java {linenos=table}
public class HiddenIterator {
    @GuardedBy("this")
    private final Set<Integer> set = new HashSet<Integer>();
    public synchronized void add(Integer i) { set.add(i); }
    public synchronized void remove(Integer i) { set.remove(i); }
    public void addTenThings() {
        Random r = new Random();
        for (int i = 0; i < 10; i++)
            add(r.nextInt());
        System.out.println("DEBUG: added ten elements to " + set);
    } 
}
```

The logging of `set` can potentially throw `ConcurrentModificationException`.

There are many others method that indirectly invoke the iteration such as
`hashCode` and `equals`.

# 2. Concurrent collections.
Many concurrent collections are introduced to replace the poor
concurrency comes from synchronized collection, including:
+ ConcurrentHashMap → Replacement for synchronized hash-based Map implementations.
+ CopyOnWriteArrayList → Replacement for synchronized List implementations (especially when traversal is dominant).
+ ConcurrentSkipListMap → Replacement for synchronized SortedMap (e.g., TreeMap wrapped with synchronizedMap).
+ ConcurrentSkipListSet → Replacement for synchronized SortedSet (e.g., TreeSet wrapped with synchronizedSet).
Concurrent collections are designed to be accessed by multiple threads and
can offer dramatic scalability improvements with little risk.

# 2.1. ConcurrentHashMap
Rather than locking all operations to the `Map` states,
`ConcurrentHashMap` uses certain techniques to enhance the scalability:
+ `Lock Striping`: Dividing the map into multiple segments (stripes) with
each being protected by a different lock, enabling multiple reads and writes as long as they don't perform on the
same bucket.
+ Improve the performance when iterating: the iteration is weakly
consistent, instead of fail-fast. Changes made to the collection
are ensured to reflect as iterating, even after `Iterator` was constructed.

This approach has the tradeoffs that it can weaken the semantics of methods
such as `size` or `isEmpty`, because the result could be out of date
by the time it is computed.

Of course, the `ConcurrentHashMap` falls behind `SynchronizedMap` in 
case you need the exclusive atomic access on the entire map.

However, `ConcurrentHashMap` is always consider a better solution for
concurrent access and should be the *drop-in* solution

## 2.2. Additional atomic map operations
Since a `ConcurrentHashMap` can not be locked for exclusive access, you can not use the `client-side` locking, like we
did with `Vector` in  [Solving vector synchronized problems](#1_1_vector_iterator_solve). If you find yourself needs 
operations, such as *put‐if‐absent*, *remove‐if‐equal*, and *replace‐if‐equal*, consider using `ConcurrentMap` instead,
which ensure atomic quality  for these operations.

## 2.3. `CopyOnWriteArrayList`
This is a replacement of `List`, offering better concurrency in some common situations and eliminating the needs to 
lock on the entire collection.

`CopyOnWriteArrayList` achieve thread safety by properly publishing the effective immutable object. Everytime the
modification occurs, the new object is created and published, allowing the old one remain immutable, which probably are
processed by other threads.

Such approach is effective if the application performs far more operations such as iteration rather than modifications. This is,
in fact, popular in notification application where a set of listeners is iterated over and being notified while the new event to 
register is fairly rare.

# 3. Blocking queues and producer-consumer pattern

Producer-consumer patterns has long been popular with key benefits:
1. Reduce code dependencies between consumers and producers.
2. Simplify workload management, consumers and producers can perform work at different and variable rates.

The class library contains several implementations of `BlockingQueue`, `LinkedBlockingQueue`, `ArrayBlockingQueue`,
and `PriorityBlockingQueue` is a priority‐ordered queue.

There is an `SynchronousQueue`, which is not really a queue at all, in that it maintains no storage space for queued elements.
This queue, though is beneficial in cases where you have multiple consumers ready for the handoff, as:
+ It reduces the latency of putting work into the queue (reduce I/O operations).
+ Allow producers to know the state of the task when it is actually handled, unlike normal queue, where producers drop the task
in a mailbox, knowing nothing about it.

# 3.1. Example: Desktop search
One type of program that is amenable to decomposition into producers and consumers is an agent that scans local drives (`DiskCrawler`)
for documents and indexes them for later searching  (`Indexer`).

Using producer-consumer patterns in this problem offers:
+ The code become more readable and reusable, each of the activities has only a single task to.
+ Yields better throughput as producers don't have to wait.


{{< code_block language="java" title="Producer and Consumer Tasks in a Desktop Search Application.">}}
public class FileCrawler implements Runnable {
    private final BlockingQueue<File> fileQueue;
    private final FileFilter fileFilter;
    private final File root;
    ...
    public void run() {
      try {
        crawl(root);
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }

    private void crawl(File root) throws InterruptedException {
        File[] entries = root.listFiles(fileFilter);
        if (entries != null) {
         for (File entry : entries)
          if (entry.isDirectory())
            crawl(entry);
          else if (!alreadyIndexed(entry))
            fileQueue.put(entry);
        }       
    }
}

public class Indexer implements Runnable {
    private final BlockingQueue<File> queue;

    public Indexer(BlockingQueue<File> queue) {
        this.queue = queue;
    }
    
    public void run() {
        try {
            while (true)
                indexFile(queue.take());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } 
    }
}
{{< /code_block >}}

Starting the Desktop search

```java {linenos=table}
public static void startIndexing(File[] roots) {
    BlockingQueue<File> queue = new LinkedBlockingQueue<File>(BOUND);
    FileFilter filter = new FileFilter() {
        public boolean accept(File file) { return true; }
    };
    for (File root : roots)
        new Thread(new FileCrawler(queue, filter, root)).start();
    for (int i = 0; i < N_CONSUMERS; i++)
        new Thread(new Indexer(queue)).start();
}
```

# 3.2. Serial Thread Confinement
To ensure thread-safety for a mutable variable, as mentioned, we can use thread confinement technique, where that object is handed off
to another thread and released from the origin thread. This is particular useful for cases where deep copy is costly.

Consider this problem: You have a database connection pool that manages a limited number of database connections, which is expensive
to be created. As such, it is better to allow worker threads, which request connections from the pool to execute database queries, 
use the connection to a particular database one by one. With thread confinement technique, you need to ensure the connection, which is mutable,
is not modified by another thread.

This can be done by using blocking queues with a little more work, it could also done with the atomic remove method of 
ConcurrentMap or the compareAndSet method of AtomicReference.

You should do a little exercise to practice this concept.

# 3.3. Deques and work stealing
