---
layout: post
title: "Fast messaging with Nats and Go"
date: 2018-10-27 21:48:29 +0200
comments: true
categories: 
 - go
 - nats
 - talks
---

This is the first of follow up serie of the talk *Fast messaging with Nats and Go* I gave at [golab 2018](golab.io). 


### The slides of the talk can be found here:

<script async class="speakerdeck-embed" data-id="001030954dc6483285e47ccf4906a7d0" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

### What is Nats
[Nats](nats.io) is a messaging system hosted under the [CNCF](www.cncf.io) umbrella, which is an *open source software foundation dedicated to making cloud native computing universal and sustainable*. It's my place-to-go while looking for software related to distributed systems. I am just a user of Nats reporting my experience and my understanding of how it works. What follows might be inaccurate or wrong :-)

### Why messaging

The way we architect and build our applications changed in the past few years, moving from monolithic apps to distributed and therefore interacting (micro) services. Together with that, some communication patterns emerged. 

Even if the most popular one is to use rest and there are popular libraries for synchronous interaction such as [grpc](https://grpc.io/) or [thrift](https://thrift.apache.org/), some interactions are better represented using events and messaging.

### Good points of messaging

Messaging is a natural way to decouple producers of messages from consumers of those messages. The producer does not know who (if any) is going to consume the message, the consumer does not know where that message is coming from.

**This makes scalability easier:** you can throw in / remove producers (or consumers) and your application will scale accordingly without having to change anything.

Messaging **enables different kinds of interaction**, for example [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html), where the state of a service is emitted as a sequence of updates which are collected in order to reconstruct the current state in another service.

Messaging makes it easier to implement all the kinds of *reactive* paradigms, where the evolution of our system is driven by the emission and the reaction to messages sent and received by our services.

### There are a lot of messaging systems, what do I need to look for in one of them?

Some aspects that we need to take in consideration are:

**Messaging patterns:** Does the messaging system support pub/sub? Request/reply? Fan-in? Any other kind of exotic messaging pattern?

**Durability of the messages:** How long will the messages last? Will be there forever(ish)? Will they expire after a short timeout? Will they be thrown away if they are not consumed immediately?

**Ordering guarantees:** Will the messages be always received in the same order the publisher is publishing them?

**Routing capabilities:** How does the system route the messages? Is it topic based? Does it support wildcarding?

**Dev / Ops experience:** This is often underestimated: how easy it is to interact with the system? How much boilerplate code is required? And also, how easy is to configure and to maintain the system? Does it require a lot of knobs to be turned in order to configure it properly?

**Performance:** Is it fast? How does it behaves under load?

**Auth / Authz:** Does it support authentication / authorization? How?

### What makes a good communication system?
The two extremes of the spectrum are

#### The Enterprise Service Bus:
Popular during the SOA movement, the bus held a lot of logic itself. Transformation, routing rules, manipulation and filtering of the messages. The result was that the logic was split between the endpoints and the bus itself, making really difficult to understand the big picture.

#### What Martin Fowler says:
In [his really popular article about microservices](https://martinfowler.com/articles/microservices.html): *"The microservice community favours an alternative approach: smart endpoints and dumb pipes"*. 

**Nats is on the dumb side**, but it's just dumb enough to be useful. 

## The protocol

The nats protocol is text based, which means that you can even telnet them to a Nats server. The messages are delimited by CR - LF .

The main messages are:

<table>
    <tr>
        <td>PUB    -</td>
        <td>Publish a message to a subject, with optional reply subject</td>
    </tr>
    <tr>
        <td>SUB   -</td>
        <td>Subscribe to a subject</td>
    </tr>
    <tr>
        <td>MSG   -</td>
        <td>Delivers a message payload to a subscriber</td>
    </tr>
    <tr>
        <td>UNSUB  - </td>
        <td>Unsubscribe from a subject</td>
    </tr>
</table>	


There are also other messages used to establish and keep the connection up, such as *CONNECT*, *PING*, *INFO* .

#### The protocol supports wildcards

Nats subjects support wildcards. The subject are tokens separated by a dot, such as **foo.bar** or **foo.bar.baz**.

There are two types of wildcards: by using **\*** a wildcard matches the token (foo.* matches foo.bar but not foo.bar.baz). If you want to match any foo.something you need to use **>** (as in **foo.>**).

## The server (or, the ops point of view)

The nats server is a (small) single Go executable. In order to run it, you just start it with

```
gnatsd -p 4222
```
and you are done. 

#### Autodiscovery
The nats server provides also a gossip based autodiscovery mechanism. By using it, you can seamlessly add and remove new nodes to the cluster to scale up / down. A new server addition is propagated to all its peers and to all the clients.

The auto discovery is enabled by setting one or more servers as *seed server* and by pointing them from the peers:

```
gnatsd -p 4222 -cluster nats://localhost:4248
gnatsd -p 5222 -routes nats://localhost:4248
```

#### All the servers form a full mesh:

{% img /images/nats/fullmesh.png %}

Each server is connected to each other, meaning that two clients will always be one hop distant **at most**. Once a client subscribes a subject, its interest will be propagated through all the servers of the mesh.

## Delivery guarantees

Nats provides very few guarantees:

- It guarantees ordering of messages.

- It's an **at most once delivery** system, which means that if the client is not connected while the message is being published, it won't receive that message. Ever. **The message is not kept on the server**, if a client looses it for some reason and wants it, some kind of recovery mechanism must be implemented (more on this later).

It's kind of like the tree falling in the forest while nobody is hearing.

## Availability

The *at most once delivery guarantee* allows Nats to be **fast and reliable**. (Almost) no state is kept on the server. 
**A slow client** is a client that is not able to consume the messages the server is sending to it for more than 1 second (the timeout is configurable). 

**The nats server behaves in a very selfish manner cutting the connection of the slow clients**, and not having to do the extra work of trying to help slow clients it achieves high levels of consistency and availability, 

## That's it for now

I'll publish the second part soon, where I'll continue continue analyzing Nats following the outline of the talk.




