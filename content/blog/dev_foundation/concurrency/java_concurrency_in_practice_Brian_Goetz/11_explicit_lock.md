+++
title = "9. Explicit lock"
date = "2025-04-30"
tags = [
    "markdown",
    "syntax",
]
+++

# 1. Lock and Reentrantlock.
`ReentrantLock` allows you put timeout on lock acquisition, offering greater thread liveness.

It also offers more flexible locking mechanism, where you can expand lock section across methods.

Following is the code example:
```java {linenos=table}
Lock lock = new ReentrantLock();
...
lock.lock();
try {
    // update object state
    // catch exceptions and restore invariants if necessary
} finally {
    lock.unlock();
}
```

You would need to lock and unlock when you enter and exit the race condition code. Users of this lock
must be wary about `unlock()` calls, because minor mistakes can cause code path never call `unlock()`, 
leaving the system hanging.

## 1.1. Polled and timed lock acquisition.
Intrinsic locks offer no built-in mechanisms to recover from locking hazards. Once a lock ordering issue arises, 
the only resolution is typically a system restart.

In contrast, explicit locks provide several recovery strategies. One such method is polling with `tryLock()`. This allows a thread to 
attempt acquiring a lock and return immediately if unsuccessful. Upon failure, the thread can then retry the acquisition or perform alternative actions like logging.

Consider the classic money transfer problem, where lock ordering can lead to deadlocks depending on the order in which accounts are given. Using the polling approach, 
if a deadlock occurs (indicated by a failure to acquire the final lock), the threads can back off and retry acquiring the locks after
a random delay, as illustrated:

```java {linenos=table}
public boolean transferMoney(Account fromAcct, Account toAcct, DollarAmount amount, long timeout, TimeUnit unit)
        throws InsufficientFundsException, InterruptedException {
    long fixedDelay = getFixedDelayComponentNanos(timeout, unit);
    long randMod = getRandomDelayModulusNanos(timeout, unit);
    long stopTime = System.nanoTime() + unit.toNanos(timeout);
    while (true) {
        if (fromAcct.lock.tryLock()) {
            try {
                if (toAcct.lock.tryLock()) {
                    try {
                        if (fromAcct.getBalance().compareTo(amount)
                                < 0)
                            throw new InsufficientFundsException();
                        else {
                            fromAcct.debit(amount);
                            toAcct.credit(amount);
                            return true;
                        }
                    } finally {
                        toAcct.lock.unlock();
                    }
                }
            } finally {
                fromAcct.lock.unlock();
            }
        }
        if (System.nanoTime() < stopTime)
            return false;
        NANOSECONDS.sleep(fixedDelay + rnd.nextLong() % randMod);
    }
}
```