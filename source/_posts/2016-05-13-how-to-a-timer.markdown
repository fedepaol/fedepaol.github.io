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

If you google for looking for --code to cut and paste--_inspiration_, you'll get a lot of results like:
- use a [countdown timer](https://developer.android.com/reference/android/os/CountDownTimer.html)
- use a dyi implementation using handlers
- be _a la mode_ and use [RxJava's timer](http://reactivex.io/documentation/operators/timer.html)

There is even a [cookbook recipe](https://androidcookbook.com/Recipe.seam;jsessionid=DF53064E03C7505C4EBF727E56E0728E?recipeId=1205) that shows how to implement it.

That might work if you had to measure the cooking time of a [portion of capellini](http://www.bettycrocker.com/how-to/tipslibrary/charts-timetables-measuring/timetable-cooking-pasta) (a very thin kind of pasta that takes no more than 6 minutes to cook).

### But wait, what if I had to bake a plum cake?
The baking time of a plum cake is longer than an hour. All the solutions I just mentioned rely on the fact that your application is running **for the whole lenght of the timer**. That could be acceptable in the desktop / server world, but it's far from acceptable in the Android context: if the app goes in background because the user wants to check his email, answer to a phone call or play a game, the operating system is likely to reclaim the resources and shutdown the app itself. 

### I can use a foreground service!
So one can start looking for a way to keep the app running in background. A [Service](https://developer.android.com/guide/components/services.html) is an Android component made specifically for this purpose. By using a Service with the [startForeground](https://developer.android.com/guide/components/services.html#Foreground) option, the chance of having it running for the whole length of the timer. 

{% img center /images/soareyoutelling_timer.jpg 300 %}

###The right way
The right way is to cheat. The idea here is to run the countdown timer as long as the app is foregrounded, showing the progress to the user _one second at the time_, but install a system alarm whenever the app goes in background. Whenever the user gets back to the app, you'll uninstall the system alarm and restart the timer from where it is supposed to start.

In this way, you'll be consuming the cpu _only when the app is in foreground_ and has all the right to consume cpu. The app won't consume a cycle when in background, apart from the effort needed to waking it up when the timer is due.

###Some code

There are three things you have to take into account:

##Running the timer in the app

##Remembering when the timer was started / how long it was supposed to run

##Handling the alarm

Conclusion: you can cheat. Always remember that the user can leave your app (but hates to burn his stufato)
Mention alarm manager / alarmclock from api 21
Commonsware wakeful 
