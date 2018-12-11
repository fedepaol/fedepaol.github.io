---
layout: post
title: "fast messaging with nats and go part 2"
date: 2018-12-11 22:32:15 +0100
comments: true
categories: 
 - go
 - nats
 - talks
---

This is the second of a follow up serie of the talk *Fast messaging with Nats and Go* I gave at [golab 2018](golab.io). 
The first part can be found [here](http://fedepaol.github.io/blog/2018/10/27/fast-messaging-with-nats-and-go/)

## What can we do with nats?

### Pub sub

As you can probably guess, given that Nats is a pure pub / sub system, we can easily implement pub / sub.

{% img /images/nats/pubsub.png 350 %}

Please remember that a nats publisher does not assume the audience. The published message can reach one, 1000 or 0 subscriber. If no subscribers are connected the message is gone forever.

### Queueing

Queueing is a variant of pub / sub where the subscriber announces itself to belong to a subscription group. In that case the Nats server will send randomly the subscribed message **only to one subscriber**. 

{% img /images/nats/queueing.png 450 %}

This is useful in case you want to implement a load balancing policy. On top of that is really easy to scale up (or down) by adding new subscribers to the subscription group.

### Request / Reply

Request / Reply is **just syntactic sugar** over the pub / sub mechanism provided by Nats: the publisher sends a reply subject together with the message and expects any reply to be sent to that reply subject.

The subscribers receive the request message together with the reply subject and sends the reply to that subject. 

{% img /images/nats/reqreply.png 450 %}

Given the nature of Nats, the request gets received to all the registered subscribers. The requestor just handles the first reply it receives and discards all the other replies.

Apart from this (quite) not efficient way to handle req / reply, this mechanism can be used together with queueing we just discussed making only one receiver of the subscription group receive the request.

## Using Nats with Go

First of all, there are clients for a lot of languages already available, from the most fancy ones (rust or elixir) to some out of fashiones (even perl!).

The go client does a bit more than just implementing the text based protocol. In particular:

#### Handles the connection / reconnection gracefully
Given that a client connected to the cluster gets notified of all the nodes of the cluster via the gossiping mechanism, whenever a client detects a disconnection from a node, it tries to reconnect to one of the other nodes of the cluster.

While the reconnection is in progress, any messages that we would like to send are buffered up to a configurable limit (the default is something like 8 Mb).

#### Tries to avoid slow client errors by emptying the socket

The client tries to receive all the available messages and pass them to the application. To do that, it buffers the messages received in order to avoid to be marked as a *slow client* by the server. When the receive buffer is full, it tries to notify the client of the library by raising errors.

### Some code

```go
    nc, _ := nats.Connect(nats.DefaultURL)

    nc.Publish("foo", []byte("Hello World"))
    nc.Subscribe("foo", func(m *nats.Msg) {
        // handle the message
    })
    nc.Flush()

    // chan subscribe
    ch := make(chan *nats.Msg, 64)
    sub, err := nc.ChanSubscribe("foo", ch)
    msg := <- ch

```

Sending and receiving takes just a few lines of code. There is also a channel based subscription where the received messages are sent through a channel.
Beware that sending are asynchronous operations, `nc.Flush()` is needed in order to force the client to send the messages out.

Using request / reply is just as straightforward:

```go
    // Requests    
    var response string
    err := c.Request("help", "help me", &response, 10*time.Millisecond)
    if err != nil {
            fmt.Printf("Request failed: %v\n", err)
    }

    // Replying
    c.Subscribe("help", func(subj, reply string, msg string) {
            c.Publish(reply, "I can help!")
    })
    // Queue group reply
    nc.QueueSubscribe("help", "workers", func(msg *Msg) {
          c.Publish(msg.Reply, "I can help!")
    })
``` 

Request / reply are blocking (whereas pub / sub are asynchronous) and as we already mentioned can be used together with subscription groups. 

The request accepts a `timeout` parameter but there is also a variant that accepts a `context` in order to let the caller be able to interrupt the request. 

### The go client does a lot more

* it provides connection lifecycle callbacks (if you want to override the reconnection mechanism)
* it provides asynchronous and synchronous versions of the apis
* it provides encoders (json and protobuf) in order to use type safe callbacks

This (and a lot more) can be found on the official nats go client [here](https://github.com/nats-io/go-nats).

I hope I triggered your curiosity in trying a powerful (yet simple) messaging system.
