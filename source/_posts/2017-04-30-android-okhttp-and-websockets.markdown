---
layout: post
title: "Android, okhttp and websockets"
date: 2017-04-30 23:04:19 +0200
comments: true
categories: 
 - android
 - rxjava
 - websocket
---
##Websockets
Rest http calls are the most common interaction between Android apps and remote servers. However, there are some scenarios where the interaction is better handled via a persistent connection: think about a chat, or a multiplayer game where data flows in both directions and the server needs to push data to the clients and to be aware of which client are connected.

This kind of scenario can be implemented through Websockets.


### OkHttp and Websockets
Given the quality of the libraries offered by Square, OkHttp was the first library I checked when I recently had to deal with websockets. Luckily, [WebSocket support was introduced in December, 2016](https://medium.com/square-corner-blog/web-sockets-now-shipping-in-okhttp-3-5-463a9eec82d1). In this post I will try to describe how to use it, and to show how it is different from using it with regular http calls. 


### Establishing the connection
Establishing the connection is pretty straightforward. You declare the OkHttp client as always:

```java

	client = new OkHttpClient.Builder()
                .readTimeout(3,  TimeUnit.SECONDS)
                .build();
```

and then take a websocket object out of it:

```java
	Request request = new Request.Builder()
                .url(serverUrl)
                .build();
        webSocket = client.newWebSocket(request, new WebSocketListener() {
							...
						});
```

Please note that by creating the websocket, OkHttp will try to establish the connection with the server. The second parameter of the ``newWebSocket`` factory method needs to implement the ``WebSocketListener`` interface, in order to get asynchronously notified of the various events occurred to the socket (such as an incoming message, or the disconnection of the socket, or a failure).


### Sending a message
Sending a message is easy. Just call ```send``` with a _String_ or a _ByteString_ as an argument. Since OkHttp will send the data using its own background thread, ```send``` can be called from any thread without worrying of blocking the current thread (and risking to get a NetworkOnMainThreadException). 

The only caveat here is that a positive result only indicates that the message was enqueued, but it does not reflect the result of the trasmission. From my understanding, the user of the library is notified only in case of failure via the ```onFailure``` callback, so an optimistic approach must be taken in place.  


### The callbacks
The [WebSocketListener](https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/WebSocketListener.java) interface provides callbacks to handle the asynchronous events related to the socket. Those includes the fact that the socket was opened (or closed), or that a new message was received.

Unlike the trasmission of the data, the interaction between the callbacks and the main Android thread needs to be implemented carefully, since ```WebSocketListener```'s method will be executed inside a background thread. Using a ```handler``` is the "vanilla Android" approach to let a background thread interact with a thread associated to a looper (such as Android's main thread).


```java
    @Override
    public void onMessage(WebSocket webSocket, String text) {
	...
        handler.sendMessage(..);
    }
```

Another way to achieve the same result would be to go reactive and expose observables to publish this events.

### Closing the connection
OkHttp provides two methods to close the connection:

## Close
```webSocket.close(0, "Bye");``` asks the server to gracefully close the connection and waits for confirmation. 
All the queued messages are trasmitted **before** closing the connection.

Since some interaction is involved, the socket might not be immediately closed. If the initialization and the closure of the connection are bound to the lifecycle of the activity (i.e. in onPause / onResume), what could happen is that some messages are received **after** close was invoked, so this needs to be handled carefully.

## Cancel
Cancel is more brutal: it just discards all the queued messages and brutally closes the socket. This has the advantage of not having to wait for the housekeeping and the trasmission of enqueued messages. However, choosing ```cancel``` over ```close``` really depends on the use case. 

# Talk is cheap, show me the code
[Here](https://github.com/fedepaol/websocket-sample) I pushed a simple example that allows an app to open the websocket when the app goes in foreground and shuts the websocket down when the app goes on background. This is the suggested approach for persistent connections. Using a service to hold the persisten connection is considered a misbehaviour and doze mode will make your app's life really hard.


The example has some weak point that could be improved:

#### Cancel is invoked when the app goes in background. 
This means that some messages could eventually get discarded. A better approach would be to invoke close and wait the connection to be gracefully closed and all the messages sent. Since in ```onPause``` the activity disposes the subscriptions, no leaking is happening. We can just hope that the application process will live long enough to let OkHttp thread to do what it needs to do in order to gracefully close the connection. A more complex approach could involve a Service or using the JobScheduler.

#### No failure of trasmission is taken into account
onFailure should listen for failures and notify the user of the failure (or even retry to send failed messages) while in the sample it just forces the disconnection.

#### No RxJava!
I wanted to keep the app simple and to avoid to introduce extra complexity, but handlers are so 2013. A better solution would have used RxJava (and probably there are many cool libraries that support that out of the box). Using RxJava would make super easy to use and transform the incoming messages and / or implement smart reconnection policies such as exponential backoff.


# Conclusion
Using websockets is a completely different beast from getting / posting to http endpoints where you fetch (or post) and forget about the call, however the OkHttp implementation is really easy to use.

On your side, you'll have not only to handle the trasmission / reception of the messages, but you will also need to monitor the state of the connection and behave accordingly. 


