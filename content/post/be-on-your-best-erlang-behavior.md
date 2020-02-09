+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Be on your best Erlang behavior"
description = "Be on your best Erlang behavior"
date = 2017-04-01T09:43:12+05:30
tags = [
    "erlang"
]
categories = [
    "erlang"
]
series = ["erlang"]
aliases = ["erlang"]
images = ["/images/gen_fsm_erlang.jpeg"]
+++

I had a great session with my team on Erlang behaviors and thought it would be worth blogging it. I have got chance to explain them about available erlang behaviors and how to choose and use them.

1.gen_supervisor
2.gen_server
3.gen_event
4.gen_fsm

When you want to employ a behavior in your applications, you should have a basic understanding of the below two types of functions.
1. API → Exposed to the user for external control and to send events.
2. Callback functions → API is a action and corresponding Callback functions produce reactions.

#### API:
For every behavior, there would be set of API’s, commonly ***start, start_link, call,send_event*** etc which will be calling the behavior modules and they in turn try to callback functions in the registered module.

The initial start or start_link API’s are responsible for calling the callback function init which is used for initialization of the behaviors to set default values.

#### Callback functions:

API and callback functions exist in pairs i.e for every API called, there should be an equivalent callback function which does the reaction and these callback functions needs to be defined in the same module(with tag ***-behavior(gen_<type>)***) as the behavior.

>Eg: Let us take gen_server as an example:
>Start gen_server by calling start or start_link API.
>You call gen_server:start API → reaches gen_server module → calls the callback function “<Your module>:init”.
Call an event on the gen_server.
>You call gen_server:call API → reaches gen_server module → calls the callback function “<Your module>:handle_call”.
----
#### gen_supervisor:
From bottom-up approach,Imagine employees in an organization managed by scrum masters, scrum masters are managed by product owners and so on up to the CEO.

When we talk from top-down, CEO is sitting at the top level and he will be called from the top level supervisor. Under him, there can be multiple supervisors and each supervisor will have some more supervisors or employees working under them. If some employee or Supervisor at some level resigns or leaves, his/her immediate supervisor should take action to fill the position to ensure workload balance.

Supervisors are employed exactly for the same purpose. supervisor and child flags decide this behavior:
if supervisor flags are given a restart_period as 3 and restart_intensity as 100, the supervisor will understand to stop restarting the child process if the process is trying to restart more than 100 times in 3 seconds.

There is also some control with child flags i.e If shutdown =1000, supervisor will not restart it if child takes more than 1000ms to start.

If restart_type is permanent, child will always be restarted or if it is temporary, child will never be restarted or if it is transient, child will only be restarted if killed by a non normal signal.

Remember that supervisor flags will always dominate child flags.

>Eg: Let us say a company is rolling out a new feature/update for its apps only to a select customers. Unfortunately, a bug was found which constantly bombard the server with connection requests. It is never good on the server side to engage such unworthy behavior. Hence supervisor stops restarting it if the restarts are too frequent. These values should be carefully adjusted based on your server capacity.

#### gen_server:
The next big generic and often used behavior in Erlang is gen_server. In simple words, it is a constantly listening process with a simple in-memory database called state. It is meant to continuously listen for synchronous or asynchronous incoming traffic and respond accordingly.

Like any other behavior, it starts with a start or start_link API which in turn calls the callback function init which fills up default state.

It has a synchronous API called “call” and an asynchronous API “cast”. The first one is expected to give a response back which is not the case for second one.

>Eg: Let us take an example scenario where you want employ gen_server for electronic voting machine. Each and every “call” will increment the vote count by one and gives back response to let the voters know whether the vote was successfully casted or not. Of course, there is a privilege where you can prevent users from the acknowledgement by using “cast” API instead of “call”,only if you believe the system is highly reliable and a vote cast may never fail and you don’t want users to perform a synchronous blocking call to handle large simultaneous vote casts.

#### gen_event:
Imagine you are trying to create a platform which is expected to add more and more features in the future.
Let us say you own a payment gateway that would accept payment requests from multiple domains like mobile prepaid recharge, online shopping, ticket vending etc. As more and more domains are added, your backend has to support them.

Gen_event behavior helps you here, it starts an event Manager process that can handle multiple kinds of events like prepaid recharge payment, ticket payment etc. We write handlers for each type of payment and add these handlers to the main event Manager process. The event Manager is like a large black-hole that assimilate in all kinds of events and broadcasts them to the event handlers. Whichever event handler matches the incoming event pattern would process the event and rest of them are discarded. Like this, you can dynamically add/remove new event handlers when your payment gateway has to support new payment domain.

>***start*** and ***start_link*** are two API’s which would call the callback function “init” for initialization of the event Manager. ***gen_event:notify*** is an API to send events to the event Manager and that event is broadcasted to all the event handlers. add_handler API is used to add new handler to the existing event Manager whereas ***delete_handler*** can be used to delete deprecated handlers.

>Remember, apart from “init” callback function for the event Manager, an “init” callback for each handler would also be called when add_handler API is triggered.

#### gen_fsm:
Imagine you have to make application that runs a elevator. An elevator is also a state machine with 4 states i.e going up,going down,floor stop and emergency stop.

{{< figure src="/images/gen_fsm_erlang.jpeg" >}}

Start a gen_fsm process with 2 empty queues inside the state. One queue(main) stores the floor numbers for the current running state i.e either going down/up and other queue(auxiliary) stores them for immediate other state i.e going up/down.

Let me explain, if you are going UP, all the key presses above the current floor will be stored in the main queue and those floor numbers are popped out one by one when the elevator reaches the numbered floor. In the meanwhile,if someone presses a lower numbered floor, the second(auxiliary) queue will be responsible for storing these numbers. When the elevator reaches the maximum numbered floor, it will load the auxiliary queue to the main queue and starts moving down.Apart from the above mentioned states, other two states are floor open(closes after a timeout)and emergency stop(stops and door stays open).

When the elevator is started, it waits for the key press in the current floor(floor open state). If the key pressed is higher than current floor, It starts moving UP(Going UP state) and all the upward key presses stored in the current running queue. During this time, if someone presses a lower numbered floor, it is stored in a auxiliary queue for the next state(Going DOWN state). While the elevator is operating normally, you can also perform an emergency STOP(emergency stop state) that stops it and releases the door lock.

#### Implementation:
>All the individual floor number key presses call a ***gen_fsm:send_event*** function that is handled by corresponding state.

>Whereas, emergency stop is special case that should be handled irrespective of the current state of the elevator. In such cases, we use ***gen_fsm:send_all_state_event*** function that is handled by a unique callback function that can alter the state of the fsm making it to halt.


