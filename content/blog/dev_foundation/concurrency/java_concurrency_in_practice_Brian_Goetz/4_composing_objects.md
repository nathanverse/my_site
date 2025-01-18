+++
title = "2. Composing objects"
date = "2025-01-15"
tags = [
    "markdown",
    "syntax",
]
+++


We don't want to have to analyze each memory access to ensure that our program is thread‐safe; we want to be able to 
take thread‐safe components and safely compose them into larger components or programs.

This chapter covers patterns for structuring classes that can make it easier to make them thread‐safe and to 
maintain them without accidentally undermining their safety guarantees.
# 1. Designing a thread-safe class

## 1.1. Gathering Synchronization Requirements
*Invariants* are defined as constraints that make a certain state of the object invalid or valid. For example, `NumberRange` instance
has two states upper and lower ranges, the lower-range value must always is lower than the upper one.

*Post-conditions* are defined as a state an object must have when it is transformed from the current state. For example, 
If the current state of a Counter is 17, the only valid next state is 18.

Your code must ensure to comply *invariants* and *post-conditions* while synchronization. If it fails to do that, the code will be left
in an inconsistent state.

For instance, as mentioned, `NumberRanger` has a multivariable invariant (by multivariable, it means the invariant involves many states
as one). You cannot update one, release and reacquire the lock, and then update the others, 
since this could involve leaving the object in an invalid state when the lock was released. 
When multiple variables participate in an invariant, the lock that guards them must be held for 
the duration of any operation that accesses the related variables.

> By using a constant field you limit the state space an object can have, which 
> is easier to deal with when it comes to synchronization.


## 1.2. State-dependent operations
An operation is said to be state-dependent when it can not be applied to the current state of an object if the object is in certain
states (*preconditions*).

For example, if you call `popElement()` when a queue is empty in a single-threading environment, it will throw error.
In multi-threading environment, however, if you dedicate a thread to wait there to `popElement` once a queue is not empty and do the task
on that element, there is chance other threads will add an element to the queue, which turn the *precondition* to `true`.

In these cases, to make thread continue to run after a condition is met, we often:
+ Use low-level mechanisms such as `wait` and `notify` (covered in chapter 14).
+ Use existing library classes, such as blocking queues or semaphores (covered in chapter 5) <- this is a more straight forward approach.

# 2. Instance confinement
This is one of the technique to ensure thread-safety for a class: implementing a thread-safe object by
confining all of its states, which are not necessarily thread-safe. All code paths are analyzed to ensure
encapsulated states are assessed with the appropriate lock held. For example

```java
@ThreadSafe
public class PersonSet {
    @GuardedBy("this")
    private final Set<Person> mySet = new HashSet<Person>();
    public synchronized void addPerson(Person p) {
        mySet.add(p);
    }
    
    public synchronized boolean containsPerson(Person p) {
        return mySet.contains(p);
    } 
}
```

From the code above, `mySet` is not thread-safe, but all methods accessing to this variable are guarded
by the intrinsic lock, making `PersonSet` a thread safe class.

However, items `Person` stored in `mySet` could be mutable, and thus making it non-thread safe. You can either
decide to make `Person` thread-safe, or guard it by a lock, a more unreliable way, and ensure 
noting other developers that any access to `Person` need to guard by that lock too.


> The Java platform class libraries offer some static methods which allow
> to turn a non thread-safe object to become a thread-safe one, such as
> `Conllections.synchronizedList`. This is done by using Decorator pattern
> to make thread-safe wrapper class for the original one.

> **⚠️ Warning:**
> Be cautious on non thread-safe states of the class, because you might accidentally publish it unsafely.

## 2.1. Java monitor pattern

This is the most basic pattern of *Instance confinement* we could imply. An object following the Java 
monitor pattern encapsulates all its mutable state and guards it with a lock.



```java
public class PrivateLock {
    private final Object myLock = new Object();
    @GuardedBy("myLock") Widget widget;
    void someMethod() {
        synchronized(myLock) {
            // Access or modify the state of widget
        }
    } 
}
```

The lock can be a private member or not. But private lock such as above code has its own advantages
+ Clients of class can't participate in its synchronization policy ‐ correctly or incorrectly:
Accidentally using lock on other same type objects are an example of a bug.
+ Verifying client-side locks are difficult, requiring examine a whole program not a class.

# 3. Delegating thread safety

```java
@ThreadSafe
public class CountingFactorizer implements Servlet {
    private final AtomicLong count = new AtomicLong(0);
    public long getCount() { return count.get(); }
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = factor(i);
        count.incrementAndGet();
        encodeIntoResponse(resp, factors);
    }
}
```