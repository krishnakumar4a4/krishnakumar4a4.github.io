+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Simplifying Erlang Beam"
description = "Simplifying Erlang Beam"
date = 2017-05-12T09:43:12+05:30
tags = [
    "erlang"
]
categories = [
    "erlang",
]
series = ["erlang"]
aliases = ["erlang"]
images = []
+++

## Light weight processes:
All the Erlang code compiles to beam code and runs on the Erlang beam virtual machine. Nevertheless to say, it is a super powerful engine that can instantly creates and runs millions of processes with each process having a very minimal footprint size of 233 words. Each erlang process is like a green thread and shares nothing with siblings.

## Erlang process scheduler:
The whole Erlang beam machine runs as a single process at the underlying operating system level just like JVM. The beam machine is an efficient virtual machine which can take control over few or all the cores of the underlying physical machine and employs its own scheduler to efficiently manage the Erlang processes running upon them. Each Erlang process is allocated a reduction budget(around 2000) and the process is only allowed to run until the whole budget is burned. The burning happens when the process is being run for every line of instruction executed. How much budget that has to burn for every instruction executing is carefully calculated and enforced in the beam machine to allow fair scheduling for all processes in the beam machine. Although you can alter this budget by changing the priority of of process I.e more priority process gets more budget.
##### Intrinsics:
You can also run more than one scheduler per each core, a scheduler is essentially a thread at the operating system level. Bringing up more schedulers per each core mean that the same core is shared between those newly come up threads.

To give a better clarity at the OS level, these scheduler threads run along with other OS level threads but doesn't exclusively take access of the cores. If there are more OS level threads sharing the same core, the beam schedulers gets less scheduling time by the operating system and hence slows down the beam machine. However, Beam machine can get more scheduling time from the operating system, if there are more schedulers started per core. But the downside is, you might accidentally add up an extra context switching burden along with 1000 other things the beam machine does. A trade-off should be carefully calculated depending on your usecase for a better performance.

Just like the Linux kernel scheduler, each beam scheduler has a run queue that holds the set of processes about to run on the scheduler. And also, a global process is run to maintain balance across the scheduler run queues. Sometimes the beam schedulers, which are threads at the OS level would jump across the cores which consume significant CPU can be stopped by configuring the beam machine itself. Turning on/off the schedulers during idle time also consumed significant CPU in older versions of beam machine.But in the latest beam machines, the schedulers are always active and this behavior is again configurable which added a huge performance gain.

## Message Passing is just copy:
When we dive deep into the internals of the beam machine, all the Erlang processes share the same heap space the operating system has allocated for the beam process. The data of each erlang process although physically exists in the same heap space but logically isolated. Hence message passing between Erlang processes is literally a copy from one logical address to another logical address and it is nothing like a network or socket or any kind of IPC mechanism. Hence it is inherently faster than any other IPC mechanism.

## Message handling:
Erlang strictly opposes the idea of shared memory and processes are communicated only with message passing. Within the beam machine, message passing is copying of message from one heap space of the process to other heap space. Whereas the message passing between two Erlang processes residing on two different physical machines happens through TCP communication. Each Erlang process has a message queue that stores all the incoming messages and does selective receive on top of it. Selective receive is only triggered whenever a new message comes into the queue. All the received messages are consumed sequentially in their order of arrival.

## Memory allocation:
The memory to the beam process is dynamically queried and allocated using a sophisticated memory allocation strategy. It itself deserves a special attention to look [at]("http://erlang.org/doc/man/erts_alloc.html").

## Concurrent GC:
[Garbage collection happens for every process using a generational mark sweep garbage collecting algorithm](https://hamidreza-s.github.io/erlang%20garbage%20collection%20memory%20layout%20soft%20realtime/2015/08/24/erlang-garbage-collection-details-and-why-it-matters.html). As the Erlang process can start and run concurrently, so is the garbage collector. The data in the heap space is divided into young heap and old heap. Young heap is frequently garbage collected where as full sweep happens when the memory to be garbage collected exceeds a certain threshold.
##### Intrinsics:
Although the GC strategy seems to be similar to other VMs. Garbage is easier and faster to collect if there are no references/shared structures. Erlang is immutable and shares nothing.

However we can also trigger garbage collection using erlang:garbage_collect function. But frequent fullsweep garbage collection may adversely impact the performance.

## Special binary handling:
Apart from the on-heap memory which is the heap memory of individual processes, there is also an off-heap memory that can store larger binaries bigger than 64KB. Hence these binaries are reference counted and hence need not be garbage collected but just neglected when all the referenced processes exits. This is an extremely efficient way to share binaries between processes and also to garbage collect large binaries.

## Faster In-memory database:
Erlang Term Storage(ETS) is faster in-memory database supported by erlang. It is basically a key-value store. ETS can hold multiple tables and they can be linked to processes and hence GC'ed when the linked process dies. There are some special provisions like providing access control to designated processes and make the table live even if the owner/parent process is dead. All the ETS tables are maintained off-heap(outside of process heap). ETS tables are light weight and as fast as the processes can access them. They also provide configurable write and read concurrency for performance intensive applications. ETS tables itself can be directly used as caching layer without looking for any third party solutions. Also, supports disc persistence if needed, but will make it bit slower.

## Distributed Database:
Mnesia is a distributed strongly consistent database. It has multiple options to allow data to be stored only on RAM(for faster access), on disc and also both. Each mnesia table is created using an erlang record structure as schema. Hence multiple similar records can be pushed into one table. All sorts of transactions, dirty operations, checkpoints can be done on this.

## Distribution:
Erlang distribution is very efficient. Each host runs a epmd(erlang port mapper daemon) process which acts as a gateway for initial handshake between erlang nodes running on physically separated hosts on a network. Every erlang node registers themselves with epmd daemon when started with distributed parameters like "-sname <shortname to node> or -name <longname to node>". If one of the erlang nodes wants to communicate with another erlang node on a different host. The request first reaches the epmd of first host and which in turn it looks for other listening epmd daemons on the network. The other epmd daemon takes the message and invoke its local erlang node. Only the initial handshake happens through the epmd and then each of the erlang nodes themselves form a direct TCP socket connection between them by choosing random ports. epmd only kicks in if the connection is lost between the erlang nodes. epmd also responsible for monitoring the health of all the nodes and broadcasts their status to their connected nodes if they are down and also tries to re-establish the connection. All these things happen under the hood and user need not be worried about the communication strategies.

##### Intrinsics:
Although there are some practical limitations of erlang distribution in scale which I have explained it in a different article of mine with a practical example - [Beauty and mystery of Erlang distribution](https://medium.com/@krishna.thokala2010/be-on-your-best-erlang-behavior-f8b478ef14cd)

## Green threads:
Erlang leverages many-to-many mapping between OS level threads and erlang level processes. Each and every thread is mapped to a erlang scheduler. Erlang processes are again mapped across the available erlang schedulers. Problems like blocking IO is mitigated by using asynchronous IO mechanisms where a OS level thread is never blocked for IO but instead process it only when the response arrives. Since the whole erlang process scheduling is being done in the user space, which is extremely faster than the scheduling at the kernel space. Inside the beam machine, scheduling is also more fairer as predicting load of a erlang process is less cumbersome because of reduction budget principle for erlang processes.

## Dirty Schedulers:
Erlang supports NIF(Natively Implemented Function)s which allows us to call the C code directly from the erlang code. But erlang doesn't have scheduling control over these C functions as reduction budget can not be estimated for a piece of C code that is loaded by user into the system. Hence lot of effort is being undergone to find a solution for this. Dirty scheduling is one among them where NIFs are suspended at regular intervals to prevent any CPU hogging.

## Error Propagation:
There are two ways spin off a new process in erlang. One is with linking and other is without linking.
When multiple process are linked in a chain, if one process terminates abnormally, all connected processes will be terminated and garbage collected instantly.
If the processes are not linked, it doesn't care about other process even if it dies.
There is also third way called monitoring, where a new process is monitored of exit signals and the parent process can take decision of what to do for itself. In this case, parent process acts as supervisor of other processes. Hence leveraging the fault tolerance and fail fast.
Error propagation is done through signals unlike message passing and is significantly faster and clean(in terms of GC).

#### Thank You!!

