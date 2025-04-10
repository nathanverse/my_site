+++
title = "5. Design a key-value store"
date = "2025-03-28"
tags = [
    "markdown",
    "syntax",
]
+++

A key-value store is a non-relational database. Each unique identifier is stored as a key with its associated value.

The key must be unique, either is a plaint text or hashed value. For performance reasons, a short key works better.

The value can be any types. It is treated as opaque object in key-value stores.

In this article, we will try to design:
+ The size of a key-value pair is small: less than 10 KB.
+ Ability to store big data.
+ High availability: The system responds quickly, even during failures.
+ High scalability: The system can be scaled to support large data set.
+ Automatic scaling: The addition/deletion of servers should be automatic based on traffic.
+ Tunable consistency.
+ Low latency.

# Single server key-value store
Designing the database which is hosted on one server is easy, but it is very quick to run out of memory. Possible solutions:
1. Compressing data.
2. Store only frequently used data in memory and the rest on disk.

But it is still not enough to handle big data access.

# Distributed key-value store.

## CAP theorem
CAP theorem states it is impossible for a distributed system to simultaneously provide more than two of these three 
guarantees: consistency, availability, and partition tolerance.

+ *Consistency*: all users see the same data at the same time no matter what node they connect to.
+ *Availability*: any client requests data should get a response even if some of the nodes are down.
+ *Partition Tolerance*: the system continues to operate despite the system is partitioned due to network failures.

Because in the real work, network failures always occur, thus the system that sacrifice this characteristic is impossible. Therefore, you
must either choose only consistency and availability.

For example, the following img illustrates the system with 3 nodes, and one node `n3` is going down.

![img](/cao.png)

In this case, if you choose consistency (like financial system which emphasizes the importance of seeing latest balance). When latest 
balance is written to `n3` but fails to propagate to other nodes before terminating, we can't write and read in other nodes, thereby
informing the service downtime and sacrificing availability.

On the other hand, to preserve availability, `n1` and `n2` allows read and write operations to be executed despite the possibility of
returning stale or incorrect data. Writing operations would be reconciled later to achieve the correct state of the system.

You should discuss with the interviewer about what characteristic you should preserve.

## System components


