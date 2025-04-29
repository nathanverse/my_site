+++
title = "8. Avoid liveness hazard"
date = "2025-04-29"
tags = [
    "markdown",
    "syntax",
]
+++

# 1. Deadlock
Deadlock can happen when threads get to wait resources held by each other. For example, in *dining philosophers* problem, deadlock can occur
when all philosophers simultaneously pick their left chopstick and wait the right stick held by the philosopher right next to them.

Thinking of *waiting resource held by another thread* as a relationship in a directed graph. Whenever there is a cycle, there are chances
deadlock can happen.

In the database system, there is mechanism to resolve deadlocks made by transactions by searching that graph and aborting the thread that 
is waiting for a long time, allowing other threads to proceed. The aborted transaction can be retried later.

JVM doesn't support resolving deadlocks, so we need to be mindful.

Deadlock can be hard to detect, as it may only happen when the system is under heavy load.

## 1.1. Lock-ordering deadlocks.
Given the following code:

```java {linenos=table}
// Warning: deadlock-prone!
public class LeftRightDeadlock {
    private final Object left = new Object();
    private final Object right = new Object();
    public void leftRight() {
        synchronized (left) {
            synchronized (right) {
                doSomething();

            } 
        }
    }
    
    public void rightLeft() {
        synchronized (right) {
            synchronized (left) {
                doSomethingElse();

            } 
        }
    } 
}
```

Two threads each call `leftRight` and `rightLeft` methods simultaneously may experience deadlock as there is cyclic locking dependency.
If you ensure locks are acquired in the same order, for example, from right -> left, then there wouldn't be deadlock. Therefore, 
lock ordering matters when write a deadlock-free code.

## 1.2. Dynamic lock order deadlocks
You should be wary that lock order may appear dynamically, like following example:

```java {linenos=table}
public void transferMoney(Account fromAccount, Account toAccount, DollarAmount amount) throws InsufficientFundsException {
    synchronized (fromAccount) {
        synchronized (toAccount) {
            if (fromAccount.getBalance().compareTo(amount) < 0)
                throw new InsufficientFundsException();
            else {
                fromAccount.debit(amount);
                toAccount.credit(amount);

            } 
        }
    } 
}
```

As you can see, lock ordering depends on the order of arguments. In case two transactions send concurrently with two account objects input
with opposite orders, deadlock can definitely happen.

One way to address this is to use hash value of the object from `Object.hashCode`. You can compare `fromAccount` and `toAccount` hash code
and order the lock acquisition in an ascending or descending order.

```java {linenos=table}
private static final Object tieLock = new Object();
public void transferMoney(final Account fromAcct,
                          final Account toAcct,
                          final DollarAmount amount)
        throws InsufficientFundsException {
    class Helper {
        public void transfer() throws InsufficientFundsException {
            if (fromAcct.getBalance().compareTo(amount) < 0)
                throw new InsufficientFundsException();
            else {
                fromAcct.debit(amount);
                toAcct.credit(amount);

            }
        }
    }
    int fromHash = System.identityHashCode(fromAcct);
    int toHash = System.identityHashCode(toAcct);
    if (fromHash < toHash) {
        synchronized (fromAcct) {
            synchronized (toAcct) {
                new Helper().transfer();

            }
        }
    } else if (fromHash > toHash) {
        synchronized (toAcct) {
            synchronized (fromAcct) {
                new Helper().transfer();

            }
        }
    } else {
        synchronized (tieLock) {
            synchronized (fromAcct) {
                synchronized (toAcct) {
                    new Helper().transfer();
                }

            }
        }
    }
}
```

However, there is case two different objects returning a same hash value. In such case, you can add a `tieLock` to ensure only one
thread does a risky action of acquiring randomly ordering locks. This approach may imply a performance suffering; however, given
a small chance of this path, it is acceptable.

In case an object has a unique, immutable, and comparable key, you can get to use it instead of hash value. You also don't need to use `tieLock`
in this case.

## 1.3. Deadlocks between cooperating objects.
Deadlocks are not always easily noticeable like two above examples, especially when the deadlock occurs in two different classes. Following
is an example

```java {linenos=table}
// Warning: deadlock-prone!
class Taxi {
    @GuardedBy("this") private Point location, destination;
    private final Dispatcher dispatcher;
    public Taxi(Dispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }
    public synchronized Point getLocation() {
        return location;
    }
    public synchronized void setLocation(Point location) {
        this.location = location;
        if (location.equals(destination))
            dispatcher.notifyAvailable(this);
    } 
}
class Dispatcher {
    @GuardedBy("this") private final Set<Taxi> taxis;
    @GuardedBy("this") private final Set<Taxi> availableTaxis;
    public Dispatcher() {
        taxis = new HashSet<Taxi>();
        availableTaxis = new HashSet<Taxi>();
    }
    
    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }
    public synchronized Image getImage() {
        Image image = new Image();
        for (Taxi t : taxis)
            image.drawMarker(t.getLocation());
        return image;
    } 
}
```

`Taxi` object stores its realtime location, and the destination it is heading towards, which get updated continually by GPS service. Meanwhile,
`Dispatcher` is like a taxi management object, it stores taxis, knows which taxis are being available through `notifyAvailable()` reported
by each taxi, and can render an image illustrating the current locations of all its taxis.

Deadlock can happen on the above code, when GPS service `setLocation` for a taxi, and the `Dispatcher` is asking to `getImage()`. This issue
in code is hard to detect, because `Taxi` instance call an alien method (in this case `Dispatcher.getImage`) while holding a lock on itself.
This type of call is called a close call.

## 1.4. Open call
The close call can cause lock ordering if the callee class, in turn, depends on the caller class. Such call also makes it hard to 
do lock-ordering analysis, because you need to look paths across classes. Therefore, open call, which is calling the alien method while
not holding lock is more favored.

Following is the solution applying open call for `Taxi` and `Dispatcher`:
```java {linenos=table}
@ThreadSafe
class Taxi {
    @GuardedBy("this")
    private Point location, destination;
    private final Dispatcher dispatcher;
    ...

    public synchronized Point getLocation() {
        return location;
    }

    public synchronized void setLocation(Point location) {
        boolean reachedDestination;
        synchronized (this) {
            this.location = location;
            reachedDestination = location.equals(destination);
        }
        if (reachedDestination)
            dispatcher.notifyAvailable(this);
    }
}


@ThreadSafe
class Dispatcher {
    @GuardedBy("this") private final Set<Taxi> taxis;
    @GuardedBy("this") private final Set<Taxi> availableTaxis;
    ...
    public synchronized void notifyAvailable(Taxi taxi) {
        availableTaxis.add(taxi);
    }
    
    public Image getImage() {
        Set<Taxi> copy;
        synchronized  (this) {
            copy = new HashSet<Taxi>(taxis);
        }
        Image image = new Image();
        for (Taxi t : copy)
            image.drawMarker(t.getLocation());
        return image;
    } 
}
```

As long as the class being called doesn't have lock-ordering, so the call to that class is.

## 1.5. Resource deadlocks
Deadlock can also happen when threads waiting on resources. For example, imagine our app has two connection pools for two different databases.
Two thread each acquire a connection from different databases and wait the connections on the other database when there is only one connection
left on each connection pool can create deadlock. Obviously, the greater size the connection pool is the less likely the deadlock occurs.

Another deadlock happens in the context of thread pool, when a task sends another task to the same `Executor` and wait for its result before
proceeding. Once all threads get to wait like that, deadlock starvation will occur and no tasks would make progress. This implies that the bounded thread pool
and interdependent tasks do not mix well together.

# 2. Avoiding and diagnosing deadlocks.
A program that never acquires more than one lock at a time cannot experience lock‐ordering deadlock, if you can try to get away with it.

You may want to document your lock ordering protocol (like using hash to determine order of lock).

Analyzing consistency of your code lock ordering by first identify the sections of code that acquire more than one lock, 
and then perform a global analysis of all such instances. If you ensure only use open calls, this process is fairly easy. Using
automated bytecode or source code analysis can be a good idea.

## 2.1. Timed lock attempts
Java support `tryLock` to allow developers put time out on the lock, when the thread fails to acquire the lock after the specified time, it
returns a failure and release the lock. You can then log what the thread is trying to do for deadlock analysis, back off, wait for a while,
possibly clearing the deadlock condition and retry the operation.

However, remember that when you acquire more than 2 locks at the same time, if deadlock occurs, releasing the outer lock may not resolve the
issue.

## 2.2. Deadlock analysis and thread dumps.
JVM provides thread dump feature to helps identify locking issues. Thread dump reports locking information, such as:
1. Which locks are held by each thread.
2. In which stack frame they were acquired.
3. Which lock a blocked thread is waiting to acquire.
4. Threads are being indirectly inconvenienced, those not involved in the deadlock cycle but blocked by waiting resource holding
by deadlock threads.

JVM does this by searching through *is-waiting-for* graph which we already mentioned.

The former java version may support more limited information in thread dump, like Java 5.0, it doesn't support dumping explicit lock.

Java version greater than 6.0 supports dumping explicit locks. You should notice that, because the nature of being associated to 
a thread and not the section of code like intrinsic lock, explicit locks when dumped may be imprecise about which stack it was acquired at.

# 3. Other liveness hazards.