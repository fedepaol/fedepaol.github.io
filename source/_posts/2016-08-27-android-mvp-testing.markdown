---
layout: post
title: "Android, MVP, dagger and testing"
date: 2016-08-27 22:52:07 +0200
comments: true
categories: 
---
### MVP is Model View Presenter
Which is a pattern that is very popular among Android developers nowdays.

I don't intend to write (yet) another guide about MVP in Android, because others have done a better job, for example:

 - [Antonio Leiva's introduction to MVP](http://antonioleiva.com/mvp-android/)
 - [Hannes Dorfmann's introduction to Mosby](http://hannesdorfmann.com/mosby/mvp/)
 - [Fernando Cejas' post on clean architecture](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)

A lot have been said about MVP (and other patterns like that), like:

 - it isolates the business logic from the UI
 - it makes faster and easier to unit test the business logic
 - it avoids having a god Fragment or Activity class that manages everything
 - it makes it easier to maintain the app

However, the first thing you notice while switching into "mvp mode", *is a great sense of order*.

In all the (few) apps I wrote before, I ended up with the well known hodgepodgey fragment or activity that contained part of the UI logic and part of the business logic.

By defining the responsabilities of the view and of the presenter with MVP, you implicitly define the interfaces between those two components (and the model), and everything fits its place.

Every touch, drag, and eventyally lifecycle events are just events that are reported back to the presenter, which then chooses what to do with them. This is powerful.

### So what is this post about?
It's about my experience with the MVP pattern, dagger (2) and testing.

Given the definition of the interface between the view and the presenter, I ingenuosly expected that testing of both (using unit tests and Espresso tests) would have been super smooth. As it often happens, the reality is quite different from what one expects and reads from blogs. In this post I will try to sum up all the issues I had during that process and the solutions I tried to put in place.

In order better illustrate the concepts, I wrote a little example that can be found on my [github repo](TODO mettere link)

The structure of the app is the same one the can be found googling for dagger / mvp , for example [here](https://github.com/antoniolg/androidmvp).

The only thing I added is a local component / module that I use in order to inject the stuff needed only buy that particular set of classes. This means that in addition to the global Component / Module classes, used to inject stuff like the storage, there will be a _local_ Component / Module used, for example, to inject the presenter into the View.

### The easy part
Is (unit) testing the presenter. The dependencies here are resolved by passing what it needs as constructor parameters.

Since no injection magic is involved here, we can just mock the view and all the other stuff the presenter needs and easily write unit tests for a Presenter instance.

This is when I started to think that it was easy.

### Testing the view

With this strong separation of roles, I expected it to be easy to mock the presenter and test the view with Espresso.

#### Injecting a mock presenter
Since the presenter is provided by the local module and injected into the view by Dagger, I had to find a way to override the Module in order to provide the mock presenter that could drive the tests.

By using the suggested way to inject the view

```java
	DaggerMainComponent.builder()
                .applicationComponent(app.getComponent())
                .mainModule(new MainModule(this))
                .build().inject(this);
```

the only way to provide a mocked presenter was to take advantage of the build variants, but I wanted to take advantage of dagger 2.
