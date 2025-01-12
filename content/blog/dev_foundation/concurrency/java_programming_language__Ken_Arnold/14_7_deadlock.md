+++
title = "4. Deadlock"
date = "2025-01-07"
tags = [
    "markdown",
    "syntax",
]
+++

Deadlock is a phenomenon where two threads attempt to acquire two different objects' locks which can never be released because the locks are being
acquired by the other thread. This is well-known concept, I just put a code where deadlocks can be occurred, if you can detect 
the sequences the deadlock can occur, you already grasped this concept well. Supposed we create two object
`Friendly`, assigned their partners, and call `hug` of each object on each thread:

```java
class Friendly {
    private Friendly partner;
    private String name;
    
    public Friendly(String name) {
        this.name = name;
    }
    
    public synchronized void hug() {
        System.out.println(
            Thread.currentThread().getName()+
            " in " + name + ".hug() trying to invoke " +
            partner.name + ".hugBack()"
        );
        partner.hugBack();
    }
    
    private synchronized void hugBack() {
        System.out.println(
                Thread.currentThread().getName()+
                " in " + name + ".hugBack()"
        );
    }
    
    public void becomeFriend(Friendly partner) {
    this.partner = partner;
    }
}
```


