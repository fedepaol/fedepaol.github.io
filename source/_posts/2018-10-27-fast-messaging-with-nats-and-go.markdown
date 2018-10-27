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
This is a follow-up post to the talk *Fast messaging with Nats and Go* I gave at [golab 2018](golab.io). On a side note, it was the widest audience I gave a talk to and I really enjoyed giving it even if it was a bit stressful on me.

### The slides

<script async class="speakerdeck-embed" data-id="001030954dc6483285e47ccf4906a7d0" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

### Nats
Nats is a messaging system hosted under the [CNCF](www.cncf.io) umbrella, which is an *open source software foundation dedicated to making cloud native computing universal and sustainable*.

### Why messaging

The way we architect and build our applications changed in the past few years, moving from monolithic apps to distributed and interacting (micro) services. Together with that, some communication patterns emerged. The most popular one is to use rest, there are rpc libraries such as [grpc](https://grpc.io/) or [thrift](https://thrift.apache.org/), but there are some interactions where using a messaging system fits more the problem we are trying to solve.

### Good points of messaging

Messaging is a natural way to decouple producers of messages from consumers of those messages. The producer does not know who (and if) is going to consume the message, the consumer does not know where that message is coming from.

**This makes scalability easier:** you can throw in / remove producers (or consumers) and your application will scale accordingly without having to change anything.

Messaging **enables different kind of interaction**, for example [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html), where the state of a service is emitted as a sequence of updates which are collected in order to reconstruct the current state in another service.

Messaging makes it easier to implement all the kinds of *reactive* paradigms, where the evolution of our system is driven by the emission and the reaction to messages sent and received by our services.

### There are a lot of messaging systems, what do I need to look for in one of them?

Some aspects that we need to take in consideration are:

##### Messaging patterns
Does the messaging system support pub/sub? Request/reply? Fan-in? Any other kind of exotic messaging pattern?
##### Durability of the messages
How long will the messages last? Will be there forever(ish)? Will they expire after a short timeout? Will they be thrown away if they are not consumed immediately?
##### Ordering guarantees
Will the messages be always received in the same order the publisher is publishing them?
##### Routing capabilities
