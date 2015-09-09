---
layout: post
title: "Mocking with Robolectric and Dagger 2"
date: 2015-09-05 08:22:47 +0200
comments: true
categories: 
---
## Why robolectric
I've been a fan of robolectric since the old days, since [when Android Studio was not an option and few developers embraced IntelliJ](http://fedepaol.github.io/blog/2012/07/23/intellij-robolectric-and-android/). I left it a bit behind after the introduction of Android Studio, since its support was far from optimal.

Things have changed, and after listening Corey Latislaw advocating its usage during [this fragmented podcast episode](http://fragmentedpodcast.com/episodes/13/) I wanted to give it a spin. Even if there is a bit of debate over its usage, mainly because tests are performed against mocked objects instead of the real framework code, it is the fastest lane to your tdd cycle because tests are run on the local jvm instead of being packed in an apk, deployed on a device and run over there. 

## Dependency Injection
One really cool thing about robolectric 3.0 is the fact that you can override the Application object declared in your manifest with a custom one (which can inherit from your application's one).

If you are using dagger (or dagger 2) and you are using the application as the source of dependency injection for your classes, this allow to easily replace your injected objects with mocks. You can even choose which mocks inject in the setup phase of your tests.

## Let's see an example:
Let's say you have your application class that exposes all the injected objects in a Dagger 2 fashion, and that you are using it to inject classes in your activities:

```java
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // stuff 
        ((MyApplication) getApplication()).getComponent().inject(this);
    }
```

Now, if we can drive the component injected within our tests, the rest of the app would use them and (hopefully) behave in a way we expect, depending on our mocks instead of the real objects.

The dependencies are provided by a module:

```java
@Module
public class ApplicationModule {
    // stuff

    @Provides
    @Singleton
    GitHubClient provideClient() {
        return new GitHubClient(mApp.getApplicationContext());
    }
    // .. Provides other stuff
}
```

`GitHubClient` is a Retrofit (2) powered client that helps to retrieve all the repos for a given user.

By using a test only application, we can provide a module from our tests. 

Let's see ApplicationModule's mocked alter ego. Note that we can override only the dependencies that we want to mock:

```java
public class MockApplicationModule extends ApplicationModule {
    List<Repo> result;
    // stuff

    @Override
    GitHubClient provideClient() {
        GitHubClient client = mock(GitHubClient.class);
        // mock behaviour
        return client;
    }

    public void setResult(List<Repo> result) {
        this.result = result;
    }
}
```

Now that everything is in place, we can use the mocked objects in our tests:

```java
@RunWith(RobolectricGradleTestRunner.class)
@Config(constants = BuildConfig.class,
        application = TestApplication.class)
public class SampleTest {
    @Before
    public void setup() {
        TestApplication app = (TestApplication) RuntimeEnvironment.application;
        // Setting up the mock module
        MockApplicationModule module = new MockApplicationModule(app);
        module.setResult(mockedResult);
        app.setApplicationModule(module);
    }
}
```

From now on, the our tested activities will be injected with our mocked github client and we will be able to test their behaviour.

## Quirks
Since the Test Application object is created before running the tests, a default application module must be provided, otherwise you'll get a dreaded NPE while running your tests.

```java
public class TestApplication extends MyApplication {
    @Override
    ApplicationModule getApplicationModule() {
        if (mApplicationModule == null) {
            return super.getApplicationModule();
        }
        return mApplicationModule;
    }}
``` 

moreover, the dependency graph is generally built inside the Application's onCreate method. Given that we want to recreate it with our mocked module, I had to add a method for that:

```java
public class MyApplication extends Application {
    // Stuff 
    @Override
    public void onCreate() {
        super.onCreate();
        initComponent();
    }

    void initComponent() {
        mComponent = DaggerRoboSampleComponent.builder()
                .applicationModule(getApplicationModule())
                .build();
    }
}
```

## Conclusion
The fact that robolectric allows you to use a custom test application object (even a different one for different tests) together with dagger is an easy way to inject mock object without having to rely on ugly setters. 

Robolectric is a fast and effective way to speed up your tdd process. All the time spent to set the tests and the mocks app is well repaid in code coverage and writing and debugging speed afterwards.

## See it in action (and have something to copy from)
[Here on github](https://github.com/fedepaol/RobolectricDependenyInjection) I put a working example that demonstrates how to inject a mocked module using robolectring.
