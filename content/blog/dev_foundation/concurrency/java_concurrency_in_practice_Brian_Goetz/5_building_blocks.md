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

![img](/vector_problem.png)

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
public static Object getLast(Vector list) {
    synchronized (list) {
        int lastIndex = list.size() - 1;
        return list.get(lastIndex);
    } 
}
public static void deleteLast(Vector list) {
    synchronized (list) {
        int lastIndex = list.size() - 1;
        list.remove(lastIndex);
    }
}
```
