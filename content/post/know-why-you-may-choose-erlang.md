+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Know why you may choose Erlang"
description = "Know why you may choose Erlang"
date = 2017-04-01T09:43:12+05:30
tags = [
    "erlang"
]
categories = [
    "erlang"
]
series = ["erlang"]
aliases = ["erlang"]
images = []
+++

When you have good understanding on Erlang, you will inherently start to see other applications differently.

>You will have tendency to ask questions like
>what is the scalability?
>What is the uptime?
>How easy is the code upgrade?
>How much availability can be guaranteed?
>Would the choice of database can match the speed of your application?
>How easy is to make your application distributed?
>Most of all, Can it crash fast and recover fast?

For all the above questions, Erlang answers it as either **high,easy and yes**.

Cut the crap and tell me what Erlang is not good for!!!

#### What is it not good for?
>Erlang is not good if your application has to interact a lot with your OS level applications.
>It is not good for file operations.
>It is not good for churning out large chunks of data like traversal, parsing etc.
>It is not good for lots of mathematical operations as it is not a general purpose language.
>It doesn’t guarantee database Consistency above 8 nodes, although I have seen good Consistency for at least 230 nodes.

Now you have some idea on capabilities and drawbacks of Erlang and you can choose wisely when to use it.

#### If you chose it, you will be paid
I might simply guess few important reasons which might led you to choose Erlang.

- Scalability
- Fault tolerance
- Distribution

Erlang supports fault tolerance by automatically and almost instantly recover from failures using the supervisor architecture. A typical Erlang application contains an initial function call to start a top level supervisor. If you are application is a group of services, then each and every service can be managed by an independent supervisor and all these supervisors can be tagged under the application’s top level supervisor. Don’t be too much confused here, the idea of supervisor is to restart the child supervisor or worker process in case of failure. There are few predefined supervisor strategies from which you can choose from based on your need from here([Be the best on your behaviors](https://medium.com/@krishna.thokala2010/be-on-your-best-erlang-behavior-f8b478ef14cd)). By just following this supervision architecture, you have added fault tolerance for your application. It is that easy.

When comes to scale, you have to embed this since you started coding the core of the application I.e individual services. In Erlang, processes are very cheap i.e as cheap as 233 words per process. Your service could be a set of functions and you should start spawning a process with erlang:spawn(function_to_be_spawned) function call wherever possible. When a function is spawned as a process, it just returns the processID which can be used to communicate to the process. Processes can only be communicated by message passing using the erlang:send(PID, “message to be sent”) function call. But on the other end, a receive call should be wrapped inside the function that is spawned with a suitable receive clause to consume the message. Remember receive is a blocking call and process would hang there until a message is received. There is also a special provision to monitor an Erlang process to get messages to the parent process when it exits with some reason. Sometimes you may need to kill the entire chain of processes if some intermediate process is exited, this is absolutely possible with zero casualties by linking each one of them using erlang:link(PID) function.

> I think the quest of scale is only satisfied with how best you can visualize your system as bunch of processes. Just like microservices.

[Behaviors in Erlang are like silver bullets that makes your life easy](https://medium.com/@krishna.thokala2010/be-on-your-best-erlang-behavior-f8b478ef14cd), there are 3 important behaviors other than the supervisor behavior described above:

1. gen_server -> use it when you want a running process that can take both synchronous and asynchronous requests and change its internal state(a small database). Eg: an user connection can be a gen_server process.
2. gen_fsm -> use it when you want to design a state machine which takes events and change its state. Eg: lock of a digital safe, for every key press, it checks whether the combination is correct and opens it and closes it after a timeout.
3. gen_event -> use it when you want to handle large amount of events and segregate them to corresponding destinations. Eg: an error logger that takes all kinds of error messages and redirects them corresponding error handler.

No application would be complete without a database. If you need to implement a simple in memory database or cache, you can use ETS(Erlang Term storage) which is hash mapped,efficient,in-memory key-value store Or If you wanted a persistent distributed database with columns, you can have Mnesia which is strongly consistent distributed database.

>Apart from all the above you have number of battle hardened libraries support from OTP(Open Telecom Platform) like SSH,SSL,INET,HTTP,FTP,SNMP,XMERL etc for almost any kind of usecase.

With scale comes distribution and Erlang handled distribution so effectively. Once two Erlang nodes on two different machines are pinged, a channel is established between the two nodes and hence calling a function or sending a message to a process on a different node is just a simple rpc:call(nodename,Module,Function,Args) or erlang:send(PID,”message”) function call respectively. Erlang makes the rest of the effort to reach the other node. If it can’t, it would let you know and you will never be hanging around for a return value. Hence it is soft realtime.

Next time you wanted to build an application that needs scale, try considering Erlang. Elixir is one of the hot programming languages to check out with Erlang capabilities and Ruby like syntax. You can try checking it out if you are comfortable with Ruby syntax.

----
#### How easy is to pick up erlang programming?

Quite easy, I would say.

Golden rule: Everything can be pattern matched inside erlang.

You need to create a module and some exports(functions that need to be called from external world).

Inside the module you can start writing number of functions. You can write multiple functions with same name and arity, they should be separated by semicolon “;” and last function of that group should be ended by dot “.”.

If you ever had a need to write a loop, you should start writing them as recursive function calls.

Within the function definition, no variable is initialized more than once and all the variables start with a Capital letter. All the statements inside function definition should be separated by comma”,”. Good news is no need of any declaration except the record datatypes which has to be declared on the top of module along with exports. Tuples({E1,E2}) are fixed size and whereas lists can be dynamic size([E1,E2]). Here E1,E2 are two variables and also elements which can hold any values like lists,tuples,records,atoms,funs etc,. Atoms start with small letter which can be used for referencing like function names,module names,table names etc,.

You can spawn a function,it turns out to be a erlang process and gives a process ID. You can use this process ID to communicate with the processes. Any process would be able to listen to messages only when it is executing a receive statement in the function from which it was spawned. A receive clause can hold a set of patterns for which input message can be pattern matched against, each and every clause is separated by a semicolon”;” and ends with a “end” keyword.

[H1|T1] = [1,2,3,4] puts 1 in H1 and [2,3,4] in T1

L1 = [H1|3,4,5,6] gives list L1 with [1,3,4,5,6] where H1 is already holding 1.

Ever need to write a conditional statement, use a “case” block. Of course “if” block also exists, but I don’t use it much because of performance reasons.

If you use emacs editor, generating boilerplate code for behaviors is very easy and you only need to concentrate on the business logic. A simple ETS table is created using function call ets:create_table and can be used as simple datastore wherever needed.

***Get Set Go for Erlang!***



