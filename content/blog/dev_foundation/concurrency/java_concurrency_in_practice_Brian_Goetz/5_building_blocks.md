+++
title = "5. Building blocks"
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

```java
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

![img](/jcip_5_vector_problem.png)

The thread calling `getClass` might throw `ArrayIndexOutOfBoundsException`,
as the `size` it obtained from the `vector` is outdated.

The same occurs for the following code, which assumes `vector.size()` will
not be changed.
```java
for (int i = 0; i < vector.size(); i++)
    doSomething(vector.get(i));
```

This occurs because we have to maintain the correct invariants between
the `size` of the vector and the underlying array of `vector` itself. To
address the problem, we can resort to `client-side` lock. For
example
```java
synchronized (vector) {
        for (int i = 0; i < vector.size(); i++)
            doSomething(vector.get(i));
}
```

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

```java

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
