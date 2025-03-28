+++
title = "1. Introduction to system scaling"
date = "2025-03-28"
tags = [
    "markdown",
    "syntax",
]
+++

# Chapter 1: Scale from zero to millions of users

## I. Database replication
Master-slave model offers several benefits
1. Better performance: Parallel processing by multiple read nodes.
2. Reliability: Data replication all over nodes.
3. High availability:  if a database is offline as you can access data stored in another database server.

In this model, when a master goes offline, a slave node will be selected and promoted to a new master.
In real life, this process can be challenging because the slave node might not have the up-to-date data.
Please refer to following materials if you want to understand techniques to solve this problem, which may be complex, [1](https://en.wikipedia.org/wiki/Multi-master_replication),
[2](https://dev.mysql.com/doc/refman/5.7/en/mysql-cluster-replication-multi-master.html).

## II. Improve latency

### Considerations for using cache

Cache is used when data is read frequently but modified infrequently.

Consistency is the caching problem that ensure to keep data in cache and data storage
in sync. When scaling across multiple regions, maintaining consistency is challenging,
for further details, please referring to “Scaling Memcache at Facebook” published by Facebook.

### Consideration using CDN

+ Cost: CDN is a third-party provider which charges you based on the amount of data transferred
  in and out. Considering to store only frequently-read data is essential to reduce cost.
+ CDN fallback: any problem with CDN should trigger fallback to load data from the origin.
+ Invalidating files: You can call CDN provided API to update or implement URL versioning for the
  data object.

### Data center
Supporting multiple data centers is crucial to improve **availability** and **user experience**
when you have users spreading across geographical areas (internationally).

Some considerations:
+ Traffic redirection: Effective tools, such as GeoDNS to direct traffic to the correct data center.
+ Data synchronization: There are 2 methods
    + Each region uses a distinct database: Limitation will lie in failover cases, where data are unavailable in another data center.
    + Replicate data across data center (more common): See [this article](https://netflixtechblog.com/active-active-for-multi-regional-resiliency-c47719f6685b) to know how Netflix implements asynchronous multi-data center replication.
+ Test and deployment: It is vital to test your app in different region as well as integrate CI/CD tools to deploy your services to multiple regions.

### Message queue
To allow each component of the system to scale independently,
we need to decouple them into different servers (called services).
Message queue is a tool enable these services communicate to each other.

When we have a message queue, producers don't need to wait for acknowledgement
from consumers to publish a message, thereby free them from being idle.

## Logging, metrics, automation

+ Logging: useful for checking logs. You can aggregate logs from each service to a
  centralized storage for easy search and viewing (DataDog is an example).
+ Metrics: Metric is use ful to check your service performance, and gain business insights.
    + Host level metrics: CPU, Memory, disk I/O, etc.
    + Aggregated level metrics: for example, the performance of the entire database tier, cache tier, etc.
    + Key business metrics: daily active users, retention, revenue, etc.
+ Automation: CI/CD is important to improve your team productivity.

## III. Database scaling

### NoSQL vs SQL
This is an excerpt from the book, non-relational databases might be the right choice if:
+ Your application requires super-low latency.
+ Your data are unstructured, or you do not have any relational data.
+ You only need to serialize and deserialize data (JSON, XML, YAML, etc.).
+ You need to store a massive amount of data.

Here is [an article](http://highscalability.com/blog/2010/12/6/what-the-heck-are-you-actually-using-nosql-for.html) that covers many use cases of NoSQL.

Readers need to dig deeper into these database architectures to gain a better understanding
of why it is the case.

### Vertical scaling
Vertical scaling comes with some serious drawback
+ Single point of failure.
+ There are hardware limits.
+ Cost is exorbitant.

### Horizontal scaling

Sharding is a strategy to distribute data to different database servers, which also distribute
traffic, and thus providing an approach to scale database. Sharding is implemented by using a hash function,
which hash based on user identity (such as ID), to determine which server will be used to populate the user's data.

We aim to have a hash function so that it can distribute data evenly across servers.

Sharding may be far more from perfect, as it introduces several problems:
1. **Resharding data**: Due to certain reasons, a shard may experience significant data growth, called shard exhaustion,
   which requires the need to use another hash function. Resharding requires you to move data around databases, which may be
   very complex. Commonly used technique is consistent hashing, which will be learned in Chapter 5.
2. **Celebrity problem**: A popular user, like Justin Bieber, may overload a server. Solving this problem might need to
   provide a shard for each celebrity.
3. **Join and de-normalization**: Joining is obviously complex if data is distributed across servers. De-normalization may be needed
   to allow query data without the need for joining.

## IV. Summary
Scaling a system is an iterative process. More fine-tuning and new strategies are needed to scale beyond millions of users.
For example, you might need to optimize your system and decouple the system to even smaller services.
