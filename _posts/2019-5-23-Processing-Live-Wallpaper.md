---
layout: post
title: 
subtitle: 
comments: true
published: false
---

### 1. What is Processing?

Processing is a hybrid language/package/IDE for learning how to code. It's particularly well-suited for a graphics environment, with the main 2 functions being `setup()` and a `draw()` loop. It's very similar to Arduino, and in-fact the IDE is almost exactly the same as Arduino's, and there are many cross-compatibilities between the two. One of the first hardware projects I ever designed was a visual graph for the Digital Input 0 pin of my Arduino. It abstracts away most of the nasty graphics programming elements, like interfacing with lower-level libraries, and allows the developer to focus on the logic of the application, and more importantly, the looks of the end result. This also means that whenever Processing is ported to a new architecture, all the underlying graphics programming has already been accomplished.

This is why I was ecstatic when I found out about [Processing for Android](https://android.processing.org/). It's available as both a desktop extension for Processing (with Android Emulator support), and as an Android app with all the features of hte desktop version (git push/pull, compile to signed apk, live wallpaper, etc).  This has been especially powerful for me, as I've tried several times to pick up Android programming, but was turned off by the UI-driven design (despite my love for GUI development, ironically). 

When I first installed Processing for Android on my phone (APDE for short), I was pleased to find Conway's Game of Life as a sample. Putting a Game of Life live-wallpaper on my Android was one of the first things I did when I got my first smartphone. It was a simple black-and-white simulation, similar to the vanilla example provided by Processing. 

### 2. What is Conway's Game of Life

The Game of Life(GoL) is a "Game" created by  British mathematician John Horton Conway in 1970, and is the quintessential example of "Cellular Automata". In short, cellular automatons are systems consisting of "cells", each having a number of states. In GoL there are only 2 states: alive and dead. With every discrete iteration, the cells change their current state based on a function of their current state, and the states of their neighbors. The GoL only has 4 simple rules:


    1. Any alive cell that is touching less than two alive neighbours dies.
    2. Any alive cell touching four or more alive neighbours dies.
    3. Any alive cell touching two or three alive neighbours does nothing.
    4. Any dead cell touching exactly three alive neighbours becomes alive.

From these 4 rules, very complex behaviors can emerge. Many different patterns or "lifeforms" have been observed, some so complex they can [self-replicate](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life#Self-replication) or even [simulate the GoL within itself](https://www.youtube.com/watch?v=xP5-iIeKXE8). It's an elegant example of how complexity can emerge from a simple set of rules.

### 3. GoL in Processing

The GoL is an example in the base distribution of Processing, so I'll include a link to it [here](https://processing.org/examples/gameoflife.html). However, it's a bit bland looking, and could use some sprucing-up. 
