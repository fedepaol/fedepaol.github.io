---
layout: post
title: "Android, MVP, Dagger and testing"
date: 2016-08-27 22:52:07 +0200
comments: true
categories:
 - android
 - testing
 - Dagger
 - MVP 
---
### MVP is Model View Presenter
.. which is a pattern that is very popular among Android developers nowdays.

I don't intend to write (yet) another guide about MVP in Android, because others have done a better job, for example:

 - [Antonio Leiva's introduction to MVP](http://antonioleiva.com/MVP-android/)
 - [Hannes Dorfmann's introduction to Mosby](http://hannesdorfmann.com/mosby/MVP/)
 - [Fernando Cejas' post on clean architecture](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)

A lot have been said about MVP (and other similar patterns), like:

 - it isolates the business logic from the UI
 - it makes faster and easier to unit test the business logic
 - it avoids having a god Fragment or Activity class that manages everything
 - it makes it easier to maintain the app

However, the first thing you notice while switching into "MVP mode", *is a great sense of order*.

In all the (few) apps I wrote before, I ended up with the well known hodgepodgey fragment or activity that contained both the UI logic and the business logic.

By defining the responsabilities of the view and of the presenter with MVP, you implicitly define the interfaces between those two components (and the model), **and everything fits its place**.

Every touch, drag, and eventyally lifecycle events are just events that are reported back to the presenter, which then chooses what to do with them. This is powerful.

## This post is about my experience with the MVP pattern, Dagger (2) and testing.

Given the definition of the interface between the view and the presenter, **I ingenuosly expected that testing of both (using unit tests and Espresso tests) would have been super smooth**. As it often happens, the reality is quite different from what one expects and reads from blogs. In this post I will try to sum up all the issues I had during that process and the solutions I tried to put in place.

In order to better illustrate the concepts, I wrote a little example that can be found on my [github repo](https://github.com/fedepaol/MVPtesting)

The structure of the app is the same one the can be found googling for Dagger / MVP , for example [here](https://github.com/antoniolg/androidMVP).

The only thing I added is a local component / module that I use in order to inject the stuff needed only by that particular set of classes. 

This means that in addition to the global Component / Module classes, used to inject stuff like the storage, there will be a _local_ Component / Module used, for example, to inject the presenter into the View.

### The easy part: testing the presenter
The dependencies are resolved by passing what it needs as constructor parameters:

```java
    @Before
    public void setup() {
        mMockView = mock(MainView.class);
        mMockStorage = mock(KeyValueStorage.class);
        mToTest = new MainPresenterImpl(mMockView, mMockStorage);
    }
```

Since no injection magic is involved here, we can just mock the view and all the other stuff the presenter needs and easily write unit tests for a Presenter instance.

Moreover, all the dependencies with external models / sources of data like retrofit can be tested by testing the behaviour of the presenter.

### The "I expected it to be easier part": testing the view

A common approach I heard around is to test the view not against a mock presenter, but against a presenter injected with mocked "external components", such as api client and storage.

What I wanted to achieve here on the other hand, is to test the view driving the behaviour of the presenter it interacts with.

With this strong separation of roles, I expected it to be easy to mock the presenter and test the view with Espresso.

#### Injecting a mock presenter
Since the presenter is provided by the local module and injected into the view by Dagger, I had to find a way to override the Module in order to provide the mock presenter that could drive the tests.

By using the common method to inject the view

```java
	DaggerMainComponent.builder()
                .applicationComponent(app.getComponent())
                .mainModule(new MainModule(this))
                .build().inject(this);
```

the only way to override the presenter since it is provided by the "real" MainModule is to use build flavours, as shown in [Android testing codelab](https://codelabs.developers.google.com/codelabs/android-testing/#0).

**However, I wanted to take advantage of Dagger 2 injecting a mock presenter.**

### The key of replacing a dependency is by overriding the Application object

By adding a factory method that returns an istance of the module in the Application class

```java
DaggerMainComponent.builder()
                .applicationComponent(app.getComponent())
                .mainModule(app.getMainModule(this))
                .build().inject(this);
```

Then we can be use a custom test runner that provides a subclass of that application object declared in the Manifest.

```java
public class EspressoTestRunner extends AndroidJUnitRunner {
    @Override
    public Application newApplication(ClassLoader cl, String className, Context context) throws
            IllegalAccessException, ClassNotFoundException, InstantiationException {
        return super.newApplication(cl, TestMvpApplication.class.getName(), context);
    }
}
```

and declare it in our gradle file

```groovy

android {
    ... 
    defaultConfig {
	...
        testInstrumentationRunner 'com.whiterabbit.windlocator.EspressoTestRunner'
	...
    }
}
```


The Application object (and its test variant) is the one responsible of providing all the modules, so by subclassing it we can drive what is provided to be injected:

```java

public class TestMvpApplication extends MvpApplication {
    private MainModule mMainModule;

    // By usint this two method we can drive whatever module we want during the tests
    // (and with that, drive what classes inject)
    @Override
    public MainModule getMainModule(MainView view) {
        return mMainModule;
    }

    public void setMainModule(MainModule m) {
        mMainModule = m;
    }
}
```

This is what the setup method would look like:

```java
@Before
public void setUp() throws Exception {
    // a mock module with the mock presenter to be injected..
    MainModule m = mock(MainModule.class);
    mMockPresenter = mock(MainPresenter.class);
    
    when(m.provideMainView()).thenReturn(mock(MainView.class)); // this is needed to fool dagger
    when(m.provideMainPresenter(any(MainView.class), any(KeyValueStorage.class)))
    	.thenReturn(mMockPresenter);
    
    Instrumentation instrumentation = InstrumentationRegistry.getInstrumentation();
    TestMvpApplication app
    	= (TestMvpApplication) instrumentation.getTargetContext().getApplicationContext();
    
    // forced to the application object
    app.setMainModule(m);
}
```

A mock module is needed to provide a mock presenter. Then the mock module is passed to the application object.
Please note that in order to have Dagger 2 working, the mock module needs to provide a view instance (even a mock one) that will never be used.

Now we can finally write a test method:

```java
    @Test
    public void testButtonClick() {
        activity.launchActivity(new Intent());
        onView(withId(R.id.main_button)).perform(click());
        verify(mMockPresenter).onButtonClicked();
    }
```

After all this struggling, we can "just test the view", meaning that we do not need to test if the mocked rest end point was called, nor if the storage was asked to write something.
**We just test the view against the presenter interface**

One piece is still missing: what if we want to test the behaviour of the view when one of its methods gets called by the presenter? In the example, the view interface offers a method to set the text displayed.

Again, one could naively think that it would be sufficient to call the method with something like 

```java
activity.getActivity().showValue("23");
```

The truth is, espresso tests run in a thread different from the UI thread. By doing that, it would result in 

    Only the original thread that created a view hierarchy can touch its views

One way to overcome this, is to call the methods in the ui thread

```java
activity.getActivity().runOnUiThread(new Runnable() { // fancy using a lambda here?
                                                 @Override
                                                 public void run() {
                                                     activity.getActivity().showValue("23");
                                                 }
                                             });
```

####Why I did not use the ```@UiThreadTest``` annotation?

Simply because it would have ended with another exception since startactivity cannot be called directly from the ui thread.

### To sum up
* Make the application provide the Module that provides the presenter(s)
* Change the testrunner in order to provide a different application
* Let the "test" application provide a mock module that provides a mock presenter
* Test!

## Conclusion

The Mvp pattern isolates the view (which needs to be as dumb as it can) from the presenter.
By instrumenting the view with a mocked presenter, you will drain those tests from any kind of logic we expect to be in the presenter. You just test that the interface between the presenter and the view is working as expected.

By doing this, you can focus on testing the business logic inside the presenter with vanilla unit tests. Your tdd loop will definetely be faster.

_A big thank as always to my proofreaders Fabio Collini & Riccardo Ciovati_
