---
layout: post
title: "Android okhttp and websockets"
date: 2017-04-16 23:04:19 +0200
comments: true
categories: 
 - android
 - rxjava
 - websocket
---
##Websockets
Rest http calls are The most common interaction between Android apps and remote servers. However, there are some scenarios where the interaction is better handled via a persistent connection: think about a chat, or a multiplayer game where data flows in both directions and the server needs to be aware of which client are connected.

Websockets offer a way to keep the connection running, and to let data flow in both directions.


### OkHttp and Websockets
Given the quality of the libraries offered by Square, OkHttp was the first library I checked when I recently had to deal with websockets. Luckily, [WebSocket support was introduced in December, 2016](https://medium.com/square-corner-blog/web-sockets-now-shipping-in-okhttp-3-5-463a9eec82d1). In this post I will try to describe how to use OkHttp with websockets, and to show how it is different from using it with regular http calls. 


### Establishing the connection
Establishing the connection is pretty straightforward. You declare the OkHttp client:

```java

	client = new OkHttpClient.Builder()
                .readTimeout(3,  TimeUnit.SECONDS)
                .retryOnConnectionFailure(true)
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

Please note that by creating the websocket, OkHttp will try to establish the connection with the server. The second parameter of the ``newWebSocket`` factory method need to implement the ``WebSocketListener`` interface, in order to get asynchronously notified of the various events occurred to the socket (such as an incoming message, or the disconnection of the socket).


### Sending a message
Sending a message is straightforward. Just call ```send``` with a String or a ByteString as an argument. Since OkHttp will send the data using its own background thread, ```send``` can be called from any thread without worrying of blocking the current thread (and risking to get a NetworkOnMainThreadException). The only caveat here is that a positive result only indicates that the message was enqueued, but it does not reflect the result of the trasmission.


### The callbacks
The [WebSocketListener](https://github.com/square/okhttp/blob/master/okhttp/src/main/java/okhttp3/WebSocketListener.java) interface provides callbacks to handle the asynchronous events related to the socket. Those includes the fact that the socket was opened (or closed), or that a new message was received.

Unlike the trasmission of the data, the interaction between the callbacks and the main Android thread needs to be implemented carefully, since WebSocketListener's method will be executed inside a background OkHttp thread. Using a ```handler``` is the suggested approach to let a background thread interact with a thread associated to a looper (such as Android's main thread).


```java
    @Override
    public void onMessage(WebSocket webSocket, String text) {
	...
        handler.sendMessage(..);
    }
```


### Going reactive
Another approach to make the callbacks interact with the main thread is by exposing observables. What I found clean was to use two different observables, one for the messages and one to notify the state of the connection.

Using relays make super easy to propagate the events to an observable that can be subscribed and then observed from a different thread.

```java
    @Override
    public void onMessage(WebSocket webSocket, String text) {
        messageRelay.accept(text);
    }
```

Moreover, in this way the events can be used as the starting point of a Rx chain to deserialize, transform and use the messages.

### Closing the connection





