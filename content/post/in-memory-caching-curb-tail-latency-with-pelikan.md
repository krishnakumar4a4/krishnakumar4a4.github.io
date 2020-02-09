+++
authors = [
    "Krishna Kumar Thokala",
]
title = "In-Memory Caching: Curb Tail Latency with Pelikan"
description = "In-Memory Caching: Curb Tail Latency with Pelikan"
date = 2016-12-24T09:43:12+05:30
tags = [
    "cache"
]
categories = [
    "cache"
]
series = ["cache"]
aliases = ["cache"]
images = []
+++

>Tail latency: When you have a bunch of servers trying to serve a request in parallel, One of the servers might take more time than others which affects the overall response time. In other words, Latency of a tail server(worst performing) is affecting the whole response time.

When thinking about scale,distribution and speed. Caching is one of the important thing that stumbles you. Understanding it, mean a lot while you wanted to improve or implement them in other programming languages. That drove me here.

### Cache Performance:
Ideally, We put the cache alongside the database to absorb the load.

If cache is effective with 95% hit-rate and if suddenly it fell down to 90%, the database hit rate would almost double. What if the cache hit-rate keeps on diminishing and most of the databases are not designed handle this kind of surge and hence the whole application performance would go haywire.

Good Cache performance = Predictable Tail Latency

### Chasing the Ghosts:
Caching in Datacenter: Machines are connected with very high speed network,Since the cost of connection is low,there would be lot of long-lived connections.

Cache: Bird’s view:
>Event-driven server: High speed and performant event driven server
>Protocol: Memcache,redis
>data Storage: Probably an In-Memory storage

***Problem***: Some Cache servers are using really high bandwidth, yet the request rate is very low.

***Digged in***: Lot of new connections are being created but very less activities on each one of them. Creating a TCP connection is at least 4+ SysCalls whereas request at max using 3 SysCalls and hence the culprit found was Connection storm.

***Problem***: Random Cache hiccups, always at the top of the hour.

***Digged in***: While logging, buffer fills up waiting for the disk and during which if some cronjob kicks in which is continuously hanging up with disk by preventing the actual buffer filled by logger to write. This is a hiccup.

***Solution***: Never do blocking IO for such logging mechanisms. A link http://faculty.salina.k-state.edu/tim/ossg/Device/blocking.html

***Problem***: Seeing several blips after each cache reboot. They never happen indefinitely and calms down.

***Digged in***: Hash table expansion happens when the key to hash entry ratio goes over a threshold i.e basically hash memory allocation doubles up. During this time, locking happens as the entries has to move over.

***Solution***: As this can’t be eliminated, clearing up cache logs might give some fresh breath to these operations.
***Problem***: Hosts with long running caches which suddenly gives oom in the kernel log and kills the cache.

***Digged in***: Fragmentation, when kernel requires contiguous memory and another application trying to get the contiguous memory will end up killing the cache process for getting hold of contiguous memory.

***Problem***: Redis instances are killed by scheduler.

***Digged in***: How much memory the kernel thinks you are using and how much memory you think you are using. These numbers may increase and this fragmentation is completely dependent on the type of load/action that is being run at the moment and it is unpredictable.

### The Mitigation Plan — Pelikan Cache:
Borrowed concept from networking, Separation of Data Plane and Control plane.

***Data Plane***: Only handles packets and handover it to Control plane.
***Control plane***: Does the networking like implementing BGP or stuff like that.

Put operations of different nature on different threads.
- Tasks which are necessary but not performance critical are placed on one thread(Control plane) like connection listening, stats aggregation, stats exporting, log dump etc.
- All the request/response processing is placed on data plane thread which is faster.
- Connects are also handled by separate data plane thread which is also faster.

Lockless Operations:
- Make stats update lockless with atomic instructions which does locking/serialization at hardware level and hence preventing software overhead.
- Make logging waitless using ring/cyclic buffer and uses atomic operations.
- Make connection hand-off lockless by using ring array.

Memory:
- Reduce internal and external fragmentation.
- Never want to do any swapping as the performance will be gone when wrote to disk.
- Avoid external fragmentation, cap all memory resources.
- Reuse connection buffers and prevent calling malloc functions and if we are able to cap all the resources means we can do preallocation as well which gives lot of runtime performance.

### Future scope of Pelikan:
##### Memcached:
- Multiple worker threads
- Binary protocol + SASL
#### REDIS:
- Rich set of data structures
- Master-slave replication
- redis-cluster
- modules
- tools

The best cache is ALWAYS FAST.

### References
https://twitter.github.io/pelikan/
https://qconsf.com/sf2016/sf2016/presentation/pelikan-quest-low-latencies.html
https://www.infoq.com/interviews/yue-twitter-pelikan-cache/
