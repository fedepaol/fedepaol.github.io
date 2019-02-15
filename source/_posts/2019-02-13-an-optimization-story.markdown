---
layout: post
title: "An optimization story"
date: 2019-02-13 23:11:49 +0100
comments: true
categories: 
 - go
---
## There are some scenarios where the order matters

Let's say that you have an api that can receive "actions" to be performed on a given endpoint. 

Think about you buying something on Amazon and then changing your mind and revoking your order. You don't want those two actions to be swapped, otherwise the cancellation will be refused (the order you want to cancel do not exists yet) and then the item you don't want anymore will be ordered (since the action of ordering the item receives the backend **after** the cancelation).

Also, in this (not so) hypothetic scenario, processing each message has a cost. For example, the message needs to be validated against some external endpoint, or it needs to be enriched with data that can be found on a database.

So here we have a dilemma: we would like to take advantage of go concurrency system and increase the throughput by processing the messages concurrently, but at the same time the order matters and you can't just pass the events to the goroutines because the order the messages are received won't be respected.

Processing the messages in a concurrent fashion would amortise the processing cost of each message. By processing the messages sequentially, a queued message will have to wait all the messages received before.

## So my question is, can we have the best of both worlds?

All this to say that I wanted to write a simple example that reorders the messages as soon as they are processed concurrently.

The basic idea is to mark each message with a sequence number, handle many messages concurrently, and pass the results to another goroutine that will then reorder the. 

The architecture is something like this:

{% img /images/optimization/fast.png 450 %}

I tried to simulate the *time consuming operation* with a random interval between 1 - 100 milliseconds:
```go
    howLong := rand.Intn(100)
    time.Sleep(time.Duration(howLong) * time.Millisecond)
```

The *consuming* part only involves printing the result of the *computation* on the standard output, in order to be able to check if the re-ordering was done properly.

### Channels and goroutines

The moving parts are:

- a [**producer** goroutine](https://github.com/fedepaol/goconcurrencylab/blob/master/fast/main.go#L67) that writes on a source channel
- many [**worker** goroutines](https://github.com/fedepaol/goconcurrencylab/blob/master/fast/main.go#L100) that process the messages sent on the source channel & write on the same output channel
- a [**result processor** goroutine](https://github.com/fedepaol/goconcurrencylab/blob/master/fast/main.go#L48) that reads the results from the output channel, try to reorder them and then print the output

## First take away lesson

This working example demonstrate how the communicating paradigm makes it easy to handle and manipulate events concurrently with Go. The fact that we do have requirements of sequentiality does not mean that we can't try to take advantage of the multiprocessing powers of Go.

### Further optimizations

During the first implementation, I kept the "reordering part" as simple as I could, meaning that the events are just appended to a slice, then the result processor keeps a "last processed" sequence number and tries to retrieve the next event to be emitted.

The code for the "draining part" is:

```go
    func drainParked(parked *[]result, next *int) {
    	for {
    		var found bool
    		for i, val := range *parked {
    			if val.seqNum == *next {
    				fmt.Printf("Processed %d\n", val.seqNum)
    				*next++
    				*parked = append((*parked)[:i], (*parked)[i+1:]...)
    				found = true
    				break
    			}
    		}
    		if !found {
    			return
    		}
    	}
}
```

I wrote the code in the first way it came to my mind, thinking that I could leverage the profiling tools provided by Go in order to see how to optimize it.



