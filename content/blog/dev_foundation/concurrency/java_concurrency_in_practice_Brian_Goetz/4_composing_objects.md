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

The `CountingFactorizer` class above delegates the thread safety responsibilities to `count`. By delegating,
we mean that the encapsulating class don't need to add any synchronized code to make it thread-safe, 
instead it resorts all to its states.

## 3.2. Independent state variables
For a class encompassing multiple states, we can delegate thread-safety to them only in case 
the encapsulating class doesn't impose any invariants on states.

For example:

```java
public class VisualComponent {
    private final List<KeyListener> keyListeners
        = new CopyOnWriteArrayList<KeyListener>();
    private final List<MouseListener> mouseListeners
        = new CopyOnWriteArrayList<MouseListener>();
    
    public void addKeyListener(KeyListener listener) {
        keyListeners.add(listener);
    }
    
    public void addMouseListener(MouseListener listener) {
        mouseListeners.add(listener);
    }
    
    public void removeKeyListener(KeyListener listener) {
        keyListeners.remove(listener);
    }
    
    public void removeMouseListener(MouseListener listener) {
        mouseListeners.remove(listener);
    }
}
```

`keyListeners` and `mouseListeners` above are two logically separate lists of listeners and thus
`VisualComponent` can delegate thread-safety to them.

## 3.3. When delegation fails
In contrast, in case the encapsulating class imposes the constraint on states, 
the code will be not thread-safe unless you use synchronization. For example

```java
public class NumberRange {
    // INVARIANT: lower <= upper
    private final AtomicInteger lower = new AtomicInteger(0);
    private final AtomicInteger upper = new AtomicInteger(0);
    public void setLower(int i) {
        // Warning -- unsafe check-then-act
        if (i > upper.get())
            throw new IllegalArgumentException(
                    "can't set lower to " + i + " > upper");
        lower.set(i);
    }
    public void setUpper(int i) {
        // Warning -- unsafe check-then-act
        if (i < lower.get())
            throw new IllegalArgumentException(
                    "can't set upper to " + i + " < lower");
        upper.set(i);
    }
    
    public boolean isInRange(int i) {
        return (i >= lower.get() && i <= upper.get());
    } 
}
```

The `NumberRange` above impose a constraint that `lower` should be lower than `upper`. The compound actions
such as `setLower` or `setUpper` potentially create a bug, because, for example in `setLower()` method
after checking `upper.get()`,  other threads can come and modify immediately `upper`, making
`lower` can have a value that is greater than `upper`.

## 3.4. Publishing underlying state variables.
A class con impose a constraint that limits the state space of its variables, like `Counter` class
requires the `count` variable to not be negative. In this case, if we publish `count` as a mutable object
, client code can make a whole instance of a class invalid.

If there is no constraint imposing, for example, imagining `temperature`, client code can change this
variable without affecting, and thus can be safely published.


# 4. Adding functionality to existing thread-safe classes.
Using the existing class or methods of that class can save effort on maintenance and testing.

In case a thread-safe class doesn't provide the method that we are expecting. A solution can be
to extend the class and use the existing methods as building blocks. For example, adding `putIfAbsent`
in the `Vector` class can be

```java
@ThreadSafe
public class BetterVector<E> extends Vector<E> {
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !contains(x);
        if (absent)
            add(x);
        return absent;
    } 
}
```

This code can be fragile, because there is the case the underlying class can change the lock on
accessing its state, making it silently not thread-safe. You should better read documents to check
lock guarantee of the author. Another disadvantage is extension will spread the implementation
to various files, making it harder to maintain.

# 4.1. Client-side locking
There is case you will not know what class to extend for integrating new functionality, especially
when factory pattern is used, for example, you will not know what subclasses of `List` will be returned
to extend.

In this case, we can add a class as a helper, as following:
```java
@ThreadSafe
public class ListHelper<E> {
    public List<E> list =
            Collections.synchronizedList(new ArrayList<E>());
    
    public boolean putIfAbsent(E x) {
        synchronized (list)  {
            boolean absent = !list.contains(x);
            if (absent)
                list.add(x);
            return absent;
        } 
    }
}
```

You must pay close attention to what lock the is using, in this case, the lock have to be used is 
`list` object which is detailed in the document of `Collections.synchronizedLis`

You might be able to see several drawbacks:
1. The same with the extension approach, lock of the used class can be changed.
2. *Client-side* locking is fragile, maintainer might use wrong lock on another object.

# 4.2. Composition
There is a less fragile alternative for adding an atomic operation to an existing class: composition, demonstrated
in the `ImprovedList` below. You extend the existing class
```java
@ThreadSafe
public class ImprovedList<T> implements List<T> {
    private final List<T> list;
    public ImprovedList(List<T> list) { this.list = list; }
    public synchronized boolean putIfAbsent(T x) {
        boolean contains = list.contains(x);
        if (contains)
            list.add(x);
        return !contains;
    }
    
    public synchronized void clear() { list.clear(); }
}
```

# 5. Documenting synchronization policies

>Document a class's thread safety guarantees for its clients; 
> document its synchronization policy for its maintainers.

Most classes don't offer any clue regarding concurrent policy, including
Java technology specifications, such as servlets and JDBC. In this case,
assuming class are thread-safe or acquire an arbitrary lock is risky.

When developing a class, you are responsible to document concurrent policy, in
a way that minimizes as many as assumptions for your colleagues and customers.

In some cases, we can imply the class is thread-safe by imaging how the
class should be implemented. Several classes, such as `ServletContext`, 
`HttpSession`, and `DataSource` are supposed to accessed by multiple threads,
therefore, authors of this code must have incorporated synchronization, or
otherwise, the code would have been reported with numerous issues.

However, be cautious of classes which are designed to store objects, such
as `ServletContext.setAttribute`. Such method might publish an object
for the entire application. Objects are published always have to be 
ensured thread-safety.

If you have no clue whether a class is thread-safe, it is always best to
assume it is not thread-safe.




