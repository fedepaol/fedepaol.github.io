---
layout: post
title: "Everything is relative"
date: 2016-04-05 06:34:46 +0200
comments: true
categories: 
 - android
 - custom views
---
####Using relative layout is bad
.. or at least, it needs to be used judiciously.
The truth is, it is _relatively_ easy to build complex layouts using RelativeLayouts, but everybody know that this comes with a cost: in order to provide that kind of flexibility, RelativeLayout does two measurement passes.
If you have nested relative layouts, you'll end up with many measurement passes that will consume your cpu. The higher the RelativeLayouts are placed in your view hierarchy, the more will be the number of measurements.

// TODO INSERIRE QUA UNA FIGURA 


But why does RelativeLayout need two passes? Given that Android is an open source framework, I could satisfy my curiosity by digging into the source code.

Let's pretend you want four children (A, B, C, D) for your layout, and the rules are the following:

A is above C
B is to right of A
D is below B and its right margin is aligned to B
C is to left of D and its top is aligned to D


In order to be able to place and measure the views correctly, the first thing RelativeLayout does is to build a graph of dependencies between the views. 
For each direction (horizontal and vertical), RelativeLayout builds a list of views that need to be measured, starting from those views that do not depend from any other view and going on with their dependants.

This explains the need for two measurement passes: the order the views need to be measured horizontally is different from the order the views need to be measured vertically.
What RelativeLayout really does here is measuring (and placing!) the children according to the other children they depend to.

In the example, the sequence of measurement are:

Horizontal:
A -> B -> D -> C

Given that while measuring horizontally the vertical size is not calculated yet, the views are temporarly assigned with the height of the parent view.

Vertical:
B -> D -> C -> A

In addition to these two measurement passes, RelativeLayout performs (some) other loops through its children to finalize the placement of those children (for example for those that are aligned to the bottom of the parent in case of wrap content).

Once the measuremnet is done, the position of the views is already calculated and for this reason the onLayout() implementation is pretty trivial.

The layout is (obviously) optimized, for example a lot of structures are being reused instead of getting recreated each time and the dependency graph is cached, but that the cache is invalidated whenver requestLayout is invoked.

### What's the take home lesson

From today, whenever I will think twice about placing a RelativeLayout inside a view, I will start thinking about all the hard work this complex piece of software needs to do behind the scene in order to offer me the flexibility I am used to.

The two measuremnet passes can be the biggest source of problems because of the risk of exponential explosion, but other than that there are a lot of computation and additional data structures involved in order to build (and maintain) those dependency lists, and a good amount of number of loops through the children in order to place them correctly. 


#### Alternatives

### GridLayout

### Custom Layouts
