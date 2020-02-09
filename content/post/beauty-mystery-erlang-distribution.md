+++
authors = [
    "Krishna Kumar Thokala",
]
title = "Beauty and mystery of Erlang distribution"
description = "Beauty and mystery of Erlang distribution"
date = 2016-08-14T09:43:12+05:30
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

I work on a network element simulator written in Erlang. For us, each network element is a bunch of Erlang processes work together to simulate a network element and we are simulating tens and hundreds of them on each Erlang node. We also run a distributed network of erlang nodes which all together renders some thousands of network elements running on one Linux machine.

Done with background!!

#### The problem:
We are not able to start more than 250 Erlang nodes on one Linux machine provided we have enough resources to start more of them, and thus limiting our network element count beyond some value.

#### Digged in:
My job was to find out why it was limited and increase it by any means possible.

I once read in Erlang system limitations document that Erlang distribution is limited by the availability of sockets on the machine.

Each socket is a combination of an IP and port. So for each IP, I can start a maximum of 65536 sockets as the number of ports(65536) is fixed on any machine.

When one Erlang node is started, one random port is chosen on the local host(127.0.0.1) which would bind to epmd port(4369) on the same localhost. This epmd daemon will be started with the first Erlang node and will share connections(ESTABLISHED) to all the other random ports on 127.0.0.1 started for each Erlang node. It would also create one more listener on a random port of 0.0.0.0.

>Each and every erlang node maintains an established connection with the host EPMD daemon which is required for active distribution. This is a default behavior of distributed protocol.

Now the mathematics, for 'n' number of erlang nodes without any distributed connectivity between them will create 'n' established connections on localhost which would occupy 'n' ports and 'n' listeners on 0.0.0.0 interface.

When we activate distribution by setting a common cookie for all and pinging on another. I could see 'n(n-1)’ established connections between all the Erlang nodes and all these utilized 'n(n-1)’ ports on the public ip of the machine.

So again mathematics, for 'n' Erlang nodes started with distribution would utilize:
>'n' ports for listeners on 0.0.0.0
>'n' ports for established connections towards epmd on localhost(127.0.0.1)
>'n(n-1)’ ports for established connections on Public Ip for distribution

As described in the Erlang system limitations document, the maximum number of erlang nodes that can be started with distribution is based on the availability of sockets and which in turn limited to 65536 for each ip.

For 'n' Erlang nodes, 'n' sockets/ports are utilized for localhost and 0.0.0.0. But 'n(n-1)’ sockets/ports are utilized on the public IP. So, the limitation would be set by fastest growing attribute from the above, I.e the maximum number of sockets supported by public IP and it is equal to 65536.

So,from the theory above we can start at most 256 Erlang nodes with distribution. But when we take out some of the low level ports below 1000 and some system utilized ports. The Erlang node count would vary between 230–256 practically.

>I am thinking to draw a network connection graph between these Erlang nodes to visualize Erlang node/port cluster. But couldn’t find an easier way to do it. I will tag that too here once done.

----

#### The mini solution:
One of the solution I could think of was to start each Erlang node with ‘-hidden’ option, so that the clustering will be linear rather than having every node connected to every other node. Node1 is only connected to node 2 and node 2 is only connected to node 3. Here node 1 to node 3 connection will be eliminated.

Hence, practically I could see only ‘2n’ established connections between nodes which would be far less than the earlier 'n(n-1)’ philosophy.

>Note: This solution should be tested practically as this may cause some problems with distributed mnesia sync and other distributed features.

Also found, inet_dist_listen_min and inet_dist_listen_max parameters as command line arguments to “erl” gives you flexibility to increase the port limit allowed for distribution.

When the erlang node count reaches beyond 200, finding a free port to start a newly distributed node will take much higher time. inet_tcp_dist module houses a mechanism to check port numbers one by one( between inet_dist_listen_min and inet_dist_listen_max) every time. As a result,all the lower port numbers are occupied first and hence increasing the time to find a higher free port number.

----

>***Disclaimer***: This article was written based on observation, I would like to refine things in the near future. Suggestions and corrections are welcome.