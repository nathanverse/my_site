+++
title = "1. Sharing Objects"
date = "2025-01-06"
tags = [
    "markdown",
    "syntax",
]
+++

# 1. Publication and escape

Making an object available to code outside its concurrent scope is call publishing. There are multiple ways
to publish an object in a class:
1. Return its reference from a non-private method.
2. Passing its reference to a method in another class.

An object that is published when it shouldn't have been is said to have escaped. Following are some examples

#### Publish an object through static fields.
```java
public static Set<Secret> knownSecrets;
public void initialize() {
    knownSecrets = new HashSet<Secret>();
}
```

#### Publish through return statement
```java
class UnsafeStates {
    private String[] states = new String[] {
        "AK", "AL" ...
    };
    public String[] getStates() { return states; }
}
```

An object being published might further publish other objects which are its non-private fields.

There is a method called `alien method`, which is the method in a class that you let other users to override, you will not know what this method
will do to your enclosed objects. This is why a contract for overriding the classes is necessary for ensuring thread-safe.

> **⚠️ Warning:**
> Once an object escapes, you have to assume that another class or thread may, maliciously or carelessly, misuse it.

Another popular escaping way is you publish an instance of the inner class, which can be used to access to the enclosing private fields.

For example:

```java
public class ThisEscape {
    public class EventListener{}
    
    public ThisEscape(EventSource source) {
        source.registerListener(
                new EventListener() {
                    public void onEvent(Event e) {
                        doSomething(e);
                    } });
    } 
}
```

## 1.1 Safe construction practices

The `ThisEscape` class above also compromise the rule that never publishes the object reference before
the construction of the object haven't been done. Because the object might be incompletely constructed.


Separate that section of code to another methods is a good option, for example


```java
public class ThisEscape {
    public class EventListener{}

    private final EventListener listener;


    public ThisEscape(EventSource source) {
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            } 
        };
    } 
    
    public void registerEventSource(EventSource source){
        source.registerListener(this.listener);
    }
}
```

# 2. Immutability
Immutability is also a great measure to ensure thread-safety, once an object is immutable, there is no concern regrading 
multiple writes from threads might cause inconsistent state. Following technique is an example.

## 2.1. Final fields
As described in section 1 in Java Programming Language blog, the following server that supports to cache factors of the last requested
values produce an atomic issue, caused by a thread read `lastFactors` while another thread attempt to modify it.
```java
@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
     private final AtomicReference<BigInteger> lastNumber
         = new AtomicReference<BigInteger>();
     private final AtomicReference<BigInteger[]>  lastFactors
         = new AtomicReference<BigInteger[]>();
     public void service(ServletRequest req, ServletResponse resp) {
         BigInteger i = extractFromRequest(req);
         if (i.equals(lastNumber.get()))
             encodeIntoResponse(resp,  lastFactors.get() );
         else {
             BigInteger[] factors = factor(i);
             lastNumber.set(i);
             lastFactors.set(factors);
             encodeIntoResponse(resp, factors);
         } 
     }
}
```

This can be addressed by the use of `immutable` objects. The idea is:
1. You group `lastNumber` and `lastFactors` into a class, which exposes no method that modify them, called `OneValueCache`
2. In the `Factorizer` class, keep a field referencing to the latest `OneValueCache`. Whenever the new factors are updated 
(because of a cache miss), new instance of `OneValueCache` created, replacing that field.

As illustrated following:

```java
@Immutable
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;
    public OneValueCache(BigInteger i,
                         BigInteger[] factors) {
        lastNumber  = i;
        lastFactors = Arrays.copyOf(factors, factors.length);
    }
    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i))
            return null;
        else
            return Arrays.copyOf(lastFactors, lastFactors.length);
    } 
}

@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
    private volatile OneValueCache cache =
            new OneValueCache(null, null);
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = cache.getFactors(i);
        if (factors == null) {
            factors = factor(i);
            cache = new OneValueCache(i, factors);
        }
        encodeIntoResponse(resp, factors);
    }
}
```

This way, the latest factors are ensured to not be altered by another thread, once the instance `OneValueCache` storing it, is referenced.
Removing the need of using lock, which can enhance the system performance, as the cost of allocation memory is typically cheaper.

# 3. Safe publication
## 3.1 Safe Publication Idioms