+++
title = "1. Synchronization"
date = "2025-01-06"
tags = [
    "markdown",
    "syntax",
]
+++

# Java Synchronization
In Java, concepts of synchronization are based on lock of objects. 
+ **Adding synchronized modifier before a method**: if the method is an instance method, the class instance's lock is acquire,
if the method is static, the java class object's of the class is acquired
+ **Using synchronized block**: Lock can be acquired on any object given

Basically, you should acquire lock on the object if its states are accessed and modified by different threads.

There are typically two types of synchronization
+ **Client-side synchronization**: Where the users of the objects, which might be shared, are responsible to 
synchronize on it, for example
    
    ```java
    public static void abs(int[] values) {
            synchronized (values) {
                for (int i = 0; i < values.length; i++) {
                    if (values[i] < 0)
                        values[i] = -values[i];
            } 
        }
    }
    ```

    In the above method, the user of values array have to add a `synchronized` block to protect it.

+ **Service-side synchronization**: Where the author of a class needs to ensure the accesses to the class's state through methods 
are synchronized.

    ```java
        public class BankAccount {
            private long number;    // account number
            private long balance;   // current balance (in cents)
        
            public BankAccount(long initialDeposit) {
              balance = initialDeposit;
            }   
        
            public synchronized long getBalance() {
              return balance;
            }   
        
            public synchronized void deposit(long amount) {
              balance += amount;
            }  
            // ... rest of methods ...
      }
  ```
  
## The need of operation atomicity.
In some cases, there could have a synchronization issue even if you are using only synchronized objects, such as atomic objects
which Java provided. Take a look at the following example, where a http server serving an endpoint, that return a factor values
of a number. The endpoint also supports a cache which return the latest factors in case the next request ask factors of
the similar value with the last request.

```java
@ThreadSafe
public class SynchronizedFactorizer implements Servlet {
    private final AtomicReference<BigInteger> lastNumber = new AtomicReference<BigInteger>();
    private final AtomicReference<BigInteger[]> lastFactors = new AtomicReference<BigInteger[]>();
    public void service(ServletRequest req, ServletResponse resp) {
      BigInteger i = extractFromRequest(req);
      BigInteger i = extractFromRequest(req);
      if (i.equals(lastNumber.get()))
        encodeIntoResponse(resp, lastFactors.get()); 
      else {
        BigInteger[] factors = factor(i); 
        lastNumber.set(i); 
        lastFactors.set(factors);
        encodeIntoResponse(resp, factors);
      }
    }
}
```

Supposed we already cache factors for a number, the following sequence of actions of two threads, thread 1 requesting factors 
for a different value, and thread 2 requesting for the same value, would cause the issue.
1. The thread 1, ask if equal to its value, turn out not -> proceed to update the cache.
2. The thread 2, passing equal condition, proceed to return the factors.
3. Thread 1 update the factors.
4. Thread 2 return the factors which was updated by thread 1.

The synchronization issue occurs as thread 2 doesn't return the correct factors. Such an issue often occurs when you need to
coordinate states of an object, in this case `lastFactors` need to be the factors of `lastNumbers`. To address this, acquire a lock
for the composition of check and return operation. Following is an example

```java
@ThreadSafe
public class CachedFactorizer implements Servlet {
    @GuardedBy("this") private BigInteger lastNumber;
    @GuardedBy("this") private BigInteger[] lastFactors;
    @GuardedBy("this") private long hits;
    @GuardedBy("this") private long cacheHits;
    public synchronized long getHits() { return hits; }
    public synchronized double getCacheHitRatio() {
        return (double) cacheHits / (double) hits;
    }
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = null;
        synchronized (this) {
            ++hits;
            if (i.equals(lastNumber)) {
                ++cacheHits;
                factors = lastFactors.clone();
            }
        }
        if (factors == null) {
            factors = factor(i);
            synchronized (this)  {
                lastNumber = i;
                lastFactors = factors.clone();
            }
        }
        encodeIntoResponse(resp, factors);
    }
}
```

Depending on the way you acquire lock, the performance of the system might vary. The above code block is actually a good
approach, as it separate the calling of long-processing method `factors = factor(i);` out of the synchronized block, allowing
thread can execute the code concurrently.

# Class synchronization documentation
As you can synchronize on any locks, this lock, fields' lock of an instance. The documentation for synchronization approaches
are essential, because:
1. If a user want to extend your class, they would meet an issue if they lock on `this`, while your class locks on
the private fields, leading to states could be accessible by multiple threads.
2. The same occurs if a user of your method needs to apply client-side synchronization.