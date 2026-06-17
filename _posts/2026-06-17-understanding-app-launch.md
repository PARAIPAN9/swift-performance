---
layout: post
title: "Understanding App Launch"
description: "App launch is our user's first experience with our app. Here's what actually happens during launch, the phases involved, and where we can influence them."
categories: [App Launch, Performance, UIKit]
---

App launch is a user experience interruption, as Apple puts it. It is the moment our user is waiting on us, before they can do anything else.

## Why it matters

Our app's launch is our user's first experience with our app, and as such it should be delightful. It is also important to test on older devices, to make sure the experience holds up across a broad set of users with different device capabilities. A launch that feels fast on the latest hardware can feel sluggish on a device that is a few years old.

## Launch types

There are three kinds of launch, and they differ by how much of our app is already resident in memory.

- **Cold:**
    - Happens after a reboot.
    - The app is not in memory.
    - No process exists. Basically, the app was not launched in a very long time.

- **Warm:** after a cold launch has happened, every kill and relaunch of the app is a warm launch.
    - The app was recently terminated.
    - The app is partially in memory.
    - No process exists.

- **Resume:** this is when the app is in the background and the user re-enters it from the home screen or the app switcher.
    - The app is suspended.
    - The app is fully in memory.
    - The process exists.

## The 400ms target

Apple says that to achieve a responsive cold launch, we should aim for roughly 400ms to render the first frame. That way we have pixels displayed to the user during the launch animation, and by the time that launch animation completes, the app is interactive and responsive.

I believe that is a very optimistic case, and it depends on the size of the app and other factors. But of course we can still find ways to reduce app launch time and aim for the lowest possible value.

To reduce app launch time, it is important to understand what happens during launch.

Launch generally starts when the user taps our icon on the home screen. Then, over the next 100 or so milliseconds, iOS does the necessary system-side work to initialize our app. That leaves us developers about 300 milliseconds to create our views, load our content, and generate the first frame.

## The phases of an app launch

The launch flows through these phases:

<div style="display:flex;flex-wrap:wrap;align-items:center;gap:8px;margin:1.5em 0;font-size:0.9em;font-weight:600;line-height:1.6;">
  <span style="background:#5856d6;color:#fff;padding:6px 12px;border-radius:8px;white-space:nowrap;">System Interface</span>
  <span style="color:#6e6e73;">→</span>
  <span style="background:#0071e3;color:#fff;padding:6px 12px;border-radius:8px;white-space:nowrap;">Runtime Init</span>
  <span style="color:#6e6e73;">→</span>
  <span style="background:#34c759;color:#fff;padding:6px 12px;border-radius:8px;white-space:nowrap;">UIKit Init</span>
  <span style="color:#6e6e73;">→</span>
  <span style="background:#ff9500;color:#fff;padding:6px 12px;border-radius:8px;white-space:nowrap;">Application Init</span>
  <span style="color:#6e6e73;">→</span>
  <span style="background:#ff2d55;color:#fff;padding:6px 12px;border-radius:8px;white-space:nowrap;">Initial Frame Render</span>
  <span style="color:#6e6e73;">→</span>
  <span style="background:#8e8e93;color:#fff;padding:6px 12px;border-radius:8px;white-space:nowrap;">Extended</span>
</div>

### 1. System Interface

DYLD, the dynamic linker, loads our shared libraries and frameworks. For an in-depth analysis, watch this presentation: [A tale of dyld, and how iOS launches your app](https://www.youtube.com/watch?v=p-dvAOHlLEc).

This is a section we can hardly influence, because it is system work. But we can influence it by avoiding dynamic library loading so we can make use of dyld3, which caches runtime dependencies and warm launches, which improves app launch performance.

### 2. Runtime Init

This is when the system initializes the Objective-C and Swift runtimes.

We cannot do much here either, except avoid static initializations and frameworks that use static initialization. But if we must use static initialization, we should consider moving code out of class load, which is invoked every time during launch, into class initialize, which is lazily invoked the first time we use a method within our class.

### 3. UIKit Init

This is when the system instantiates our `UIApplication` and our `UIApplicationDelegate`. For the most part, this is system-side work: setting up event processing and integration with the system. But we can still affect this phase if we subclass `UIApplication` or do any work in `UIApplicationDelegate` initializers.

### 4. Application Init

This is where we can have the most impact on our app launch time. Nowadays everyone is using `UIScene`s, so it is important to create our view controllers only in `scene(_:willConnectTo:options:)` and not also in `application(_:didFinishLaunchingWithOptions:)`. Doing both leads to a common pitfall that results in performance losses and, likely, unpredictable bugs in our code base.

### 5. Frame Render

This one is relatively straightforward. This is when we create our views, perform layout, and then draw them: `loadView`, `viewDidLoad`, `layoutSubviews`.

We can affect this phase by reducing the number of views in our hierarchy. We can do that by flattening our views to use fewer of them, or by lazily loading views that are not shown during launch. We should also take a look at our Auto Layout and see if we can reduce the number of constraints we are using.

### 6. Extended

This is the app-specific period from our first commit until we show our final frame to the user. This is when we load the asynchronous data. Not every app has this phase. During it, the app should be interactive and responsive.

## How do we know we need improvements?

How do we know we need to make our app launch faster? We of course need to measure first the app launch under certain conditions and in a certain states. I will leave that topic for another article, because I wanted to keep this one short and concise.

Thanks for reading. If you found it helpful, you can share it with others.

## Useful link

- [WWDC19 - Optimizing App Launch](https://developer.apple.com/videos/play/wwdc2019/423/)
