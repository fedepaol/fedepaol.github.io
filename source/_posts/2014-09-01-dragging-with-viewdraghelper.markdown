---
layout: post
title: "Dragging with ViewDragHelper"
date: 2014-09-01 20:48:20 +0200
comments: true
categories: 
 - android
 - viewdraghelper
 - mozilla
---

While working on my [last endeavour](https://bugzilla.mozilla.org/show_bug.cgi?id=909434), a patch to Firefox for Android to allow the urlbar to be dragged in order to show content hidden behind the main view, I had to deal with ViewDragHelper and understand how it works.

The final result (please note that the patch is still under review) is something like this:

{% youtube rKRIZB6nQfg %}

It caused me more than one headache, and for this reason I am writing this post hoping it might be helpful to anybody wanting to tinker with it.

ViewDragHelper's usage is not well documented, but [this post](http://flavienlaurent.com/blog/2013/08/28/each-navigation-drawer-hides-a-viewdraghelper) by Flavien Laurent is the best place you could start from.

In order to provide a simpler example for this post, I'll introduce a simplified version of what I have done on Firefox, without all the extra code needed to interact with the rest of the app.

Let's start with..

##How touch events are handled
A good source of information is the [official documentation](http://developer.android.com/training/gestures/viewgroup.html). However, I'll write a short introduction here.

Whenever a touch event happens, the parent view is being asked if it wants to handle that event in place of its children. This is done by calling its ``onInterceptTouchEvent()`` method, which should return true if the parent view wants to handle the event.  

In case the event is trapped by the parent, its ``onTouchEvent()`` method gets called and it must return true if the event is handled.

Children view can also rebel against their parent tiranny, and disable this mechanism by calling ``requestDisallowInterceptTouchEvent()``. By doing that, they ensure that the touch event wont be passed to the parent view.

{% img /images/touches.png 350 350 'touches' 'images' %}

##How does ViewDragHelper work
The idea behind it is pretty simple. You register a draghelper on a container view

```java
	mDragHelper = ViewDragHelper.create(this, 1.0f, new DragHelperCallback());
```

and then you set a couple of entry points, one to listen if a drag is being started (or is in progress), the other to handle the motion events and perform the dragging when the event is being passed to the view it registered against:
```java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        if (mDragHelper.shouldInterceptTouchEvent(event)) {
                return true;
        }
        return super.onInterceptTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mDragHelper.processTouchEvent(event);
        return true;
    }
```

ViewDragHelper will be asked to check if a motion event is part of a dragging process. The behaviour of the whole dragging process is ruled by a `DragHelperCallback` instance passed on creation.
`DragHelperCallback` has method that need to be implemented to be notified of particular evens, such as:

* a change in the dragging state
* a change in the dragged view location
* when and where the dragged view was released

It also has methods used to influence the dragging behaviour:  

* clamp the position of the view / the range it can be dragged within
* check whether a particular subview can be dragged

A whole drag process is intended a sequence of `Down` / `Move` / `Up` events over a particular target view.
Whenever a drag process starts, ViewDragHelper finds the topmost child that contains the location of the motion event, and asks us if that particular view can be dragged or not in `tryToCaptureView()` method.  
This is *more or less* the theory involved in the dragging process. On top of that, ViewDragHelper offers also `settleAt` methods to let the views settle to their rest location gracefully.

Since explaining in words it's not the easiest thing (nor I am particularly good to explain), I'll introduce the simplified app I used to understand (a bit) how ViewDragHelper works.

#Enters DragQueen

{% img https://farm6.staticflickr.com/5128/5356147569_686637006e.jpg 350 350 'dragqueen' 'images' %} [Licensed under commons creative](https://www.flickr.com/photos/kbyte/5356147569/in/photolist-9aiE4T-oZtA-6YfGh-ebJmp8-jCeRGT-7xRKgx-7csUvo-nFTXP8-nFTXNg-9DAWnB-dbWyHz-m2FLMf-6x5QY2-5vkGLh-9jwh4a-KCBSK-dJTFb-4XRtDY-ouiHNw-od7U9J-oiKnUm-6DTq9a-6d1uXV-ocLauF-7VXBD4-n4y95v-ouj2w5-oaD6SK-owCHq3-6xQaS8-je4k6-ng25wJ-oDWV9V-os7FZk-8oeWRF-oRT83M-2PtLZZ-6x5Pkt-49vCUN-49rtdk-S9buU-f5Wr7L-49vCQw-nVMgN7-6wABHo-fX91x-6x5PQH-51DtKg-7mVAco-miQ2Q/)

(Just kidding). DragQueen is a (ultra) simplified version of what I implemented on fennec with a button named queen that you can drag. 

{% youtube Vo4381SSNn0 %} 

It consists of:

+ OuterLayout (the root element of our activity, the one that contains the views we want to drag)
+ a front panel which can be dragged

{% img /images/draghelper.png 350 350 'draghelper' 'images' %}

To make the things a bit more complex we want to enable the dragging only from a particular subview, Queen. To make the things even more complex, we want to be also able to interact with Queen button while the dragging is not happening.

We also allow only two rest locations, so if the view is released mid-way it will settle to its open / close location depending on the speed and the location of when the view is released.
Finally, note that OuterLayout contains also a button that is hidden when main layout is in its closed state.

### OuterLayout
Outerlayout is a ViewGroup that extends a RelativeLayout.  
As I wrote before, the two methods ViewDragHelper needs to hook into are

```java
    @Override
    public boolean onInterceptTouchEvent(MotionEvent event) {
        if (isQueenTarget(event) && mDragHelper.shouldInterceptTouchEvent(event)) {
                return true;
        } else {
            return false;
        }
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        if (isQueenTarget(event) || isMoving()) {
            mDragHelper.processTouchEvent(event);
            return true;
        } else {
            return super.onTouchEvent(event);
        }
    }
```

You may notice that `onInterceptTouchEvent` if has another condition. This is because we want to drag mainlayout only if the touch targets the Queen (it would not be drag-queen otherwise). This is a simplified version of what happens in Fennec, where we want to intercept the drag only if it starts from the toolbar to avoid to interfere with the web content.

In any case, checking if Queen is targeted is quite easy:

```java
    private boolean isQueenTarget(MotionEvent event) {
        int[] queenLocation = new int[2];
        mQueenButton.getLocationOnScreen(queenLocation);
        int upperLimit = queenLocation[1] + mQueenButton.getMeasuredHeight();
        int lowerLimit = queenLocation[1];
        int y = (int) event.getRawY();
        return (y > lowerLimit && y < upperLimit);
    }
```

##Other methods that influence the behaviour of the dragging are:
####tryCaptureView
```java
	@Override
        public boolean tryCaptureView(View view, int i) {
            return (view.getId() == R.id.main_layout);
        }
```

which gives draghelper the permission to drag main layout). You *must* return true up there for the view you want to be dragged.

####getViewVerticalDragRange && clampViewPositionVertical (there are *Horizontal* too)
```java
        public int getViewVerticalDragRange(View child) {
            return mVerticalRange;
        }

        @Override
        public int clampViewPositionVertical(View child, int top, int dy) {
            final int topBound = getPaddingTop();
            final int bottomBound = mVerticalRange;
            return Math.min(Math.max(top, topBound), bottomBound);
        }
```
which do what you expect them to do, setting limit for the dragging. In this particular case, vertical range is set to half the size of screen.

### DragQueen 

Note also how ```mMainLayout``` is set as clickable with ```android:clickable="true"```. This prevents touch events to be passed down to the view below when it is closed..

---

## Callbacks
There are several callbacks you will want to implement in order to react to the events related to the dragging:

####onViewDragStateChanged
```java
	@Override
        public void onViewDragStateChanged(int state) {
            if (state == mDraggingState) { // no change
                return;
            }
            if ((mDraggingState == ViewDragHelper.STATE_DRAGGING || mDraggingState == ViewDragHelper.STATE_SETTLING) &&
                 state == ViewDragHelper.STATE_IDLE) {
                // the view stopped from moving.

                if (mDraggingBorder == 0) {
                    onStopDraggingToClosed();
                } else if (mDraggingBorder == mVerticalRange) {
                    mIsOpen = true;
                }
            }
            if (state == ViewDragHelper.STATE_DRAGGING) {
                onStartDragging();
            }
            mDraggingState = state;
        }
```
notifies the state transitions of DragHelper between `DRAGGING`, `IDLE` or `SETTLING` state.


####onViewPositionChanged
```java
	public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {
```
whose purpouse is pretty clear.

####onViewReleased
```java
        @Override
        public void onViewReleased(View releasedChild, float xvel, float yvel) {
            final float rangeToCheck = mVerticalRange;
            if (mDraggingBorder == 0) {
                mIsOpen = false;
                return;
            }
            if (mDraggingBorder == rangeToCheck) {
                mIsOpen = true;
                return;
            }
            boolean settleToOpen = false;
            if (yvel > AUTO_OPEN_SPEED_LIMIT) { // speed has priority over position
                settleToOpen = true;
            } else if (yvel < -AUTO_OPEN_SPEED_LIMIT) {
                settleToOpen = false;
            } else if (mDraggingBorder > rangeToCheck / 2) {
                settleToOpen = true;
            } else if (mDraggingBorder < rangeToCheck / 2) {
                settleToOpen = false;
            }

            final int settleDestY = settleToOpen ? mVerticalRange : 0;

            if(mDragHelper.settleCapturedViewAt(0, settleDestY)) {
                ViewCompat.postInvalidateOnAnimation(OuterLayout.this);
            }
        }
```
is where you (_might_) want to let the view go into its rest place. I made it behave in such way that dragging speed (and direction) is more important than the place you are releasing the view.

---

## Bonus methods

```java
	mDragHelper.settleCapturedViewAt(0, settleDestY))
```

is a helper method that will make your view smoothly settle at the given destination.

---

## Quirks and reasons for headaches

#### ViewDragHelper sets the offset of the target view ..
  .. by calling ``offsetTopAndBottom``, which is ok *but* you have to remember that a layout round called by any of the children of outerLayout (or the parent view you are passing to the draghelper) will reset that offset. What you are going to see in that case is your dragged view getting back at its rest position.

A possibile solution to this is to force back the parent where it was before:
``` java
	mMainLayout.addOnLayoutChangeListener(new View.OnLayoutChangeListener() {
            @Override
            public void onLayoutChange(View v, int left, int top, int right, int bottom, int oldLeft, int oldTop, int oldRight, int oldBottom) {
                if (mOuterLayout.isMoving()) {
                    v.setTop(oldTop);
                    v.setBottom(oldBottom);
                    v.setLeft(oldLeft);
                    v.setRight(oldRight);
                }
            }
        });
```

#### ViewDragHelper always want to intercept the top most child in z order
If you have some view in between, but you want to be able to drag a lower one, you have to let ViewDragHelper think that your view is the topmost one.
```java
        @Override
        public int getOrderedChildIndex(int index) {
            int mainLayoutIndex = indexOfChild(mMainLayout);
            if (index > mainLayoutIndex) {
                return mainLayoutIndex;
            } else {
                return index;
            }
        }
```

---


#What's more

ViewDragHelper offers a lot more features than those I just presented. *DragQueen* implements only vertical dragging, but you can drag your views horizontally too. Again, refer to the excellent post by Flavien for more details.

Moreover, ViewDragHelper allows you to intercept drag events that start from the edge of the screen, which is the way its used in the `NavigationDrawer`.

All it needs to enable it is to call 

```java 
mDragHelper.setEdgeTrackingEnabled(ViewDragHelper.EDGE_LEFT);
```
and to implement 
```java
@Override
public void onEdgeDragStarted(int edgeFlags, int pointerId) {
	mDragHelper.captureChildView(mMainLayout, pointerId);
}
```


#TL;DR
ViewDragHelper is a bit complex and as I said before not well documented. However it allows you to drag views around with very little code, and it can be used to implement nice effects. In any case you can ~~unrestrainedly copy~~ _take inspiration_ from DragQueen source code on [GitHub](https://github.com/fedepaol/dragqueen) (it seems to work). 
I really hope this post does not contain too many errors and that you enjoyed reading it as much as I did writing.

If you liked this post, consider following me on twitter [@fedepaol](https://twitter.com/fedepaol)

