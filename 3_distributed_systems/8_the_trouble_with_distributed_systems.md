### Chapter 8: The Trouble with Distributed Systems

The last chapter considered some things that can go wrong with distributed databases. This chapter generalises that to distributed _systems_. In a distributed system, effectively anything can go wrong. The chapter focuses on networks (because "distributed" effectively means "networked") and clocks (because clocks appear to offer a silver-bullet solution to distributed system problems. Spoiler alert: both can and will fail.

### Faults and Partial Failures
Software on a single computer is predictable, because when it fails it will by default crash the whole system. Not counting software bugs, internal faults are abstracted away to "it crashed". It is neither possible nor desirable to do the same in a distributed system. Instead, we have to confront _partial failure_, which is much more unpredictable and opaque than failures on a single computer.

### Cloud Computing and Supercomputing
Two different ways to scale: horizontally and vertically. Vertical scaling of a web service means running it on something like a supercomputer. The benefit here is that you can let everything crash, but the downside is that it's much more expensive to set up and scale (plus you generally can't use cloud services). If you're using a cloud instead of a supercomputer, you'll have a much higher rate of partial failure.

Because of this, cloud systems are attempts to build a reliable system from unreliable components. This seems weird - why wouldn't a system be only as reliable as its weakest link? - but is actually a standard computing idea. TCP is an example of building a reliable packet-delivery system on top of a less-reliable protocol (IP). You can sacrifice a bit of speed to lower the chance of failure (e.g. by retrying, or by transmitting extra error-correcting codes).

### Unreliable Networks
Distributed systems are "shared-nothing", which means they only communicate over the network and not by sharing memory or disk. Communicating over the network means sending packets, which is a bit like shouting into the void: a sent packet may never arrive, or may arrive very late, or out-of-order. If you don't get a response to your request, you don't know whether the request is stuck in a queue and will execute later, or whether it's totally failed. From the sender's perspective, it's reasonable to set a timeout and only wait a fixed quantity of time for a response - but even if you don't get a response, the request may still be delivered.

### Network Faults in Practice
How common are network faults, really? Very common. 12 per month in a medium-sized DC, both hardware failures and human error. Very many things can cause a network fault, from switch software upgrades to sharks biting undersea cables.

This section is a long attempt to convince the reader that _you have to handle network faults_, because they will happen and, left unhandled, can cause arbitrarily bad things to happen.

### Detecting Faults
Distributed systems generally need to detect if a node is faulty. A load balancer needs to stop sending requests to unhealthy nodes, and a db needs to tell if a leader dies so it can promote a new leader. How can you do this if any network request can fail? Sometimes you'll get useful info even in the presence of a failure (e.g. if nothing's running on the port, the OS will send you back a RST/FIN packet). You can't count on this, though. In practice you're forced to set a timeout and assume it's dead after the timeout expires (and handle the case where it comes back to life thinking nothing is wrong).

### Timeouts and Unbounded Delays
How long should a timeout be? As usual, it's a tradeoff: short timeouts identify faults faster, but are more likely to incorrectly declare a node dead. Declaring a node dead risks duplicating an action if it wakes up, or increasing load on the system if it's actually serving requests and helping out. The problem is that network delays can be indefinite in length - minutes, hours, even days.

Most (not all) delays are due to queueing. There are lots of different queues in a network. The network switch holds a queue for when multiple nodes are sending packets to the same destination. The OS on the destination machine queues packets if the serving program is busy (on the socket). When a VM is paused waiting for CPU, the VM monitor will queue incoming data in a buffer. TCP sockets will queue packets before sending sometimes as well.

When a system is operating under high load, these queues will build up. Under light load, most of these queues will be invisible.

### Synchronous vs Asynchronous networks
