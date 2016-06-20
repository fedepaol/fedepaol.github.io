---
layout: post
title: "Reactive Android timers, countdowns and lifecycle"
date: 2016-06-20 23:06:59 +0200
comments: true
categories: 
 - android
 - lifecycle 
---

#### Ok, I must confess
the title is built to draw people's attention, because you know, nowdays everything is done in a reactive fashion. RxJava is superhelpful, but **if we forget the ecosystem our apps are running into**, we risk to forget the _proper_ way to implement certain tasks in Android. 


#### Why do we need a whole post about timers?
Recently, I had to implement a countdown timer in Android.

If you google for ~~code to cut and paste~~ _inspiration_, you'll get a lot of results like:

 - use a [countdown timer](https://developer.android.com/reference/android/os/CountDownTimer.html)
 - use a dyi implementation using handlers
 - be _a la mode_ and use [RxJava's timer](http://reactivex.io/documentation/operators/timer.html)

There is even a [cookbook recipe](https://androidcookbook.com/Recipe.seam;jsessionid=DF53064E03C7505C4EBF727E56E0728E?recipeId=1205) that shows how to implement it.

That might work if you had to measure the cooking time of a [portion of capellini](http://www.bettycrocker.com/how-to/tipslibrary/charts-timetables-measuring/timetable-cooking-pasta) (the fastest cooking pasta I could think of).

### But wait, what if I had to bake a plum cake?
Baking a plum cake takes longer than an hour. All the solutions I just mentioned rely on the fact that your application is running **for the whole lenght of the timer**. 

This could be acceptable in the desktop / server world, but it's far from acceptable in the Android context: if the app goes in background because the user wants to check his email, answer to a phone call or play a game, **the operating system is likely to reclaim the resources and shutdown the app itself**. In any case, the device will turn off after a short time. If you think that using a [wakelock](https://developer.android.com/training/scheduling/wakelock.html) will solve the problem... it will, but the user won't be happy of all the battery wasted by the screen.

### I can use a foreground service!
So one can start looking for a way to keep the app running in background. A [Service](https://developer.android.com/guide/components/services.html) is an Android component made specifically for this purpose. 

By using a Service with the [startForeground](https://developer.android.com/guide/components/services.html#Foreground) option, your app will stay alive through the whole lenght of the timer. When the timer is finished, it just has to throw a notification and a broadcast so the user will know that the timer expired.

{% img center /images/soareyoutelling_timer.jpg 300 %}

This approach will work, but it has a drawback. Your app (or let's say at least the service) needs to be running for the whole length of the timer. This is a waste of memory and cpu.

###The right way
The right way is to take advantage of what the OS offers. The idea here is to run the countdown timer as long as the app is foregrounded, showing the progress to the user _one second at the time_, but set a system alarm whenever the app goes in background. Whenever the user gets back to the app, you'll cancel the system alarm and restart the timer from where it is supposed to start.

Here what it would look like when the user gets back to the app before the timer is expired (on the left) and when the timer expires while the app is in background (on the right):

{% img /images/timer_resume.png 300 %} {% img /images/timer_pause.png 300 %}

From the user's perspective, the timer is running even if the app is in background, because whenever he returns to the app he sees what he is expecting to see (the time passed). On the other hand, if the timer expires when the app is in background, a friendly notification will remind him that he has to take the plum cake out of the oven.

Inside the app however, the timer will run **only when the app is in foreground** and has all the rights to consume cpu because the user is using the app.


###Some code

A simplified version of what I am describing can be found in my [github repo](https://github.com/fedepaol/AndroidTimerSample)

There are three things you have to take into account:

##Running the timer in the app
This is the easiest part: you can use a countdown timer, a handler, rxjava or whatever you want to ~~copy and paste~~ take inspiration from.
In my example I'll use a countdown timer since it's simple to use and servers the purpouse.

```java
private void startTimer() {
    mCountDownTimer = new CountDownTimer(mTimeToGo * 1000, 1000) {
	    public void onTick(long millisUntilFinished) {
		mTimeToGo -= 1000;
		updateTimeUi();
	    }
	    public void onFinish() {
		mState = TimerState.STOPPED;
		onTimerFinish();
		updateTimeUi();
	    }
    }.start();
}
```

##Remembering when the timer was started / how long it was supposed to run
This is the trickiest part.
In the example I store the starting time inside the shared preferences storage. It will persist even if the app is killed.

```java
            mPreferences.setStartedTime(getNow());
```

That value is used when resuming the app in order to check how long the timer has to run (or if the time did expire):

```java
    private void initTimer() {
        long startTime = mPreferences.getStartedTime();
        mTimeToGo = (TIMER_LENGHT - (getNow() - startTime));
        if (mTimeToGo <= 0) { // TIMER EXPIRED
            mTimeToGo = TIMER_LENGHT;
            onTimerFinish();
        } else {
            startTimer();
        }
    }
```

The app tries to retrieve the start time value. If there still is  some time to run, the countdown restarts for the remaining length of time. Otherwise the timer is reset and the user is notified of the timer expiration.

Please note that this is a ultra simplified version that assumes that _the timer is running_. The [github sample](https://github.com/fedepaol/AndroidTimerSample) checks also if the timer was started or not.

##Handling the alarm

This is simple. You should set the alarm that triggers a broadcast receiver through the alarm manager:

```java
    @Override
    protected void onPause() {
        super.onPause();
        long wakeUpTime = (mPreferences.getStartedTime() + TIMER_LENGHT) * 1000;
        AlarmManager am = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
        Intent intent = new Intent(this, TimerExpiredReceiver.class);
        PendingIntent sender = PendingIntent.getBroadcast(this, 0, intent, PendingIntent.FLAG_CANCEL_CURRENT);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            am.setAlarmClock(new AlarmManager.AlarmClockInfo(wakeUpTime, sender), sender);
        } else {
            am.set(AlarmManager.RTC_WAKEUP, wakeUpTime, sender);
        }
    }
```

and canel it in onResume:

```java
    @Override
    protected void onResume() {
        super.onResume();
        Intent intent = new Intent(this, TimerExpiredReceiver.class);
        PendingIntent sender = PendingIntent.getBroadcast(this, 0, intent, PendingIntent.FLAG_CANCEL_CURRENT);
        AlarmManager am = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
        am.cancel(sender);
   }
```

Launching a system notification that brings the user back to the app when clicked is trivial. You can check the [sample on github](https://github.com/fedepaol/AndroidTimerSample).


##Conclusion
What I wrote today may sound obvious to a lot of experienced developers.

However, I thought it was a post worth writing since it's a good example of how you should always remember the ecosystem your app is being run into.
If you forget this and think that **your app is the most important app the user has in his phone**, you'll face some unexpected behaviours (the app gets killed) or you will piss the user off (the app needs to be active for the whole length of the timer).

