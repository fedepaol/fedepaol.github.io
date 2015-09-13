---
layout: post
title: "Testing RxJava (and Dagger 2)"
date: 2015-09-13 21:26:15 +0200
comments: true
categories: 
 - android
 - rxjava
 - testing
---
## Testing RxJava
While catching up with the latest Android novelties I could not ignore RxJava, which seems to grow in popularity between android developers.

If you just heard about it, and you want to get your feet wet, I really recommend Dan Lew's [Grokking with RxJava](http://blog.danlew.net/2014/09/15/grokking-rxjava-part-1/) series as a starting point.

**RxJava is asynchronous by nature**, so unit testing it might seem a daunting at first, especially if you use that asynchronous interaction to test stuff. Luckily, RxJava (and RxAndroid) come with a couple of tools that will make our life a lot easier. 

## What to (unit) test
There are at least a couple of things you'll want to test:

1. You will want to test the **observables**, meaning not only the observables you built, but also the resulting composition of the various operators you may want to apply to them.
2. Given a certain observable (or its mock), you will want to test **how the rest of your application behaves while triggered by the subscription**.

## Testing the observables
Despite the fact that a subscription is asynchronous, there are (at least) a couple of ways to make the stream of your observable synchronous.

The first way is by using 
```Java
ResultToCheck res = myObservable.toBlocking().first();
```

This works because [toBlocking](http://reactivex.io/RxJava/javadoc/rx/Observable.html#toBlocking%28%29) converts the observable to a blocking one, while [first](http://reactivex.io/documentation/operators/first.html) returns the first emitted element.
The calling code will wait synchronously until the observer calls onCompleted().

**The official way to test an observable** is by using a [TestSubscriber](http://reactivex.io/RxJava/javadoc/rx/observers/TestSubscriber.html), an helper subscriber provided directly by the RxJava framework.
As with toBlocking, a test subscription is synchronous. 
Here you can find an example:

```Java
Observable<RubberChicken> obs = obsFactory.getObservable();
TestSubscriber<RubberChicken> testSubscriber = new TestSubscriber<>();
obs.subscribe(testSubscriber);

testSubscriber.assertNoErrors();
List<RubberChicken> chickens = testSubscriber.getOnNextEvents();
// Assert your chickens integrity here
```

`TestSubscriber` comes with a bunch of helper methods for testing, like specific assertions and other stuff. On top of that, its `getOnNextEvents()` method is blocking and  will return all the emitted items as elements of a list.
This is a neat way to test not only your observers, but also to check if the compositions you put in place are working as expected. That makes testing observables super easy.

## Testing the subscription
Once your observables are in place, you will likely to be observing them on some thread, and subscribing them on some other thread. This will make it harder for us to test how our activity (or fragment) reacts to a triggered subscription. 

RxJava (and RxAndroid) provide a way to override the schedulers exposed when `Schedulers.io()` or `AndroidSchedulers.mainThread()` are called. By replacing them with `Schedulers.immediate()`, your code will run immediately and you'll be able to see its results.

The solution is a bit hacky, since we need to call `reset()` method before overriding RxJava's schedulers, which is package protected. I _took inspiration_ from Alexis Mas' [blogpost](http://alexismas.com/blog/2015/05/20/unit-testing-rxjava/) extending RxJavaPlugins class (there no need for that with RxAndroid):
```Java
package rx.plugins;

public class RxJavaTestPlugins extends RxJavaPlugins {
    RxJavaTestPlugins() {
        super();
    }

    public static void resetPlugins(){
        getInstance().reset();
    }
}

```

Registering a scheduler hook that provides a custom implemetation (Schedulers.immediate()) will end up in overriding the schedulers we are using.
This is how the `setup()` and `teardown()` methods will look like (here I am using robolectric but it makes no difference with AndroidTests):

```Java
@RunWith(RobolectricGradleTestRunner.class)
@Config(constants = BuildConfig.class,
application = TestRobolectricApplication.class)
public class SubscriberTest {
    private void forceImmediateScheduler() {
        RxJavaTestPlugins.resetPlugins();
        RxAndroidPlugins.getInstance().registerSchedulersHook(new RxAndroidSchedulersHook() {
            @Override
            public Scheduler getMainThreadScheduler() {
                return Schedulers.immediate();
            }
        });

        RxJavaPlugins.getInstance().registerSchedulersHook(new RxJavaSchedulersHook() {

            @Override
            public Scheduler getIOScheduler() {
                return Schedulers.immediate();
            }
        });
    }

    @Before
    public void setUp() {
        forceImmediateScheduler();
    }

    @After
    public void tearDown() {
        RxJavaTestPlugins.resetPlugins();
    }}
    /* Your tests here */
}
```

This, together with a non blocking observable (for instance by replacing your long taking observable with a mocked `Observable.just()`) will make our test synchronous.

In order to inject a mocked observable, we can override the Application object used by Robolectric,  as described in my [previous post here](http://fedepaol.github.io/blog/2015/09/05/mocking-with-robolectric-and-dagger-2/) .

## Bonus point: debugging
If the unit tests are not enough, and you want to check what is happening inside the chaining / transformation of the stream, you can set an `ObservableExecutionHook` that will be triggered when observables are being called:

```Java
   private void enableRxTrack() {
        RxJavaPlugins.getInstance().registerObservableExecutionHook(new DebugHook(new DebugNotificationListener() {
            final String TAG = "RXDEBUG";
            public Object onNext(DebugNotification n) {
                Log.v(TAG, "onNext on " + n);
                return super.onNext(n);
            }


            public Object start(DebugNotification n) {
                Log.v(TAG,"start on "+n);
                return super.start(n);
            }


            public void complete(Object context) {
                super.complete(context);
                Log.v(TAG,"oncomplete n "+context);
            }

            public void error(Object context, Throwable e) {
                super.error(context, e);
                Log.e(TAG,"error on "+context);
            }
        }));
    }
```

#TL;DR:
- Use TestSubscriber when testing how an observable (or a composition of observables) behaves
- Mock your observable and override the default schedulers to test how the subscribing class works
- Enable the tracking of your observables by registering an observable execution hook

A working example (rubber chickens included) can be found on my [github repo](https://github.com/fedepaol/TestingRxJava).

### References
* [Unit testing rxjava (observables)](https://medium.com/ribot-labs/unit-testing-rxjava-6e9540d4a329) by Iván Carballo
* [Unit testing rxjava (subscription)](http://alexismas.com/blog/2015/05/20/unit-testing-rxjava/) by Alexis Mas
* [This](http://fragmentedpodcast.com/episodes/3/) and [this](http://fragmentedpodcast.com/episodes/4/) episodes of [Fragmented Podcast](http://fragmentedpodcast.com) where Dan Lew gave some insights about RxJava, where I heard about the scheduler overriding trick  

