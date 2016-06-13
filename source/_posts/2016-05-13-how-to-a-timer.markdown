---
layout: post
title: "How to a implement a timer, the Android way"
date: 2016-05-13 23:06:59 +0200
comments: true
categories: 
 - android
 - lifecycle 
---

#### Why do we need a whole post about timers?
Recently, I had to implement a countdown timer in Android.

If you google for _inspiration_, you'll get a lot of results like:
- use a [countdown timer](https://developer.android.com/reference/android/os/CountDownTimer.html)
- use a dyi implementation using handlers
- be _a la mode_ and use [RxJava's timer](http://reactivex.io/documentation/operators/timer.html)

There is even a [cookbook recipe](https://androidcookbook.com/Recipe.seam;jsessionid=DF53064E03C7505C4EBF727E56E0728E?recipeId=1205) that shows how to implement it.

That might work if you had to measure the cooking time of a [portion of capellini](http://www.bettycrocker.com/how-to/tipslibrary/charts-timetables-measuring/timetable-cooking-pasta) (a very thin kind of pasta that takes no more than 6 minutes to cook).

### But wait, what if I had to bake a plum cake?
The baking time of a plum cake is longer than an hour. All the solutions I just mentioned rely on the fact that your application is running **for the whole lenght of the timer**. That could be acceptable in the desktop / server world, but it's far from acceptable in the Android context: if the app goes in background because the user wants to check his email, answer to a phone call, or play a game, your app is likely to be killed, and the timer with it.

### I can use a foreground service!
A more viable option would be to install one of the mentioned implementation inside a service that requests to run on foreground. That would extend the chance of your app to live for the whole lenght of the timer. 

A lot of results on google. you have the activity, launch a handler, or a countdown timer, or a thread

A service.

The right way

Conclusion: you can cheat. Always remember that the user can leave your app (but hates to burn his stufato)
Mention alarm manager / alarmclock from api 21
Commonsware wakeful 
