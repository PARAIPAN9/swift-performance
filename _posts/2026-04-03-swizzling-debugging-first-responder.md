---
layout: post
title: "Swizzling: Debugging the First Responder Chain"
description: "Method swizzling lets you intercept any Objective-C method at runtime. Here's how I used it to trace the full first responder chain, to understand a strange focus behaviour."
categories: [Debugging, Runtime, UIKit]
---

Sometimes a bug lives entirely in behaviour you cannot set a breakpoint on. The first responder chain is one of those places. UIKit moves focus across responders silently: no delegate, no notification by default, no easy way to know who called `becomeFirstResponder` or when.

This is a case where method swizzling is the right tool.

## What swizzling is

Swizzling is a runtime technique from the Objective-C world. Every Objective-C method is just an entry in a method table, a mapping from selector to function pointer. The runtime exposes `method_exchangeImplementations`, which swaps two entries in that table.

After the swap, calling the original selector runs your replacement function. Calling the replacement selector runs the original function. This is the core trick that makes swizzling work.

In Swift, you can only swizzle methods that are visible to the Objective-C runtime. That means methods on `NSObject` subclasses, or methods annotated with `@objc`.

`UIResponder` is an `NSObject` subclass, so every method on it, including `becomeFirstResponder` and `resignFirstResponder`, is swizzleable.

## The strange behaviour

I was investigating some unexpected focus behaviour in an app that embeds a `WKWebView`. Things were not working as expected: focus was moving in a way that did not match what the code was doing, and I could not tell which part of the hierarchy was actually first responder at any given moment.

The tricky part with `WKWebView` is that it manages its own internal view hierarchy, including `WKContentView`, a private WebKit view that actually holds the first responder when the web content is focused. You cannot inspect this easily, and there is no delegate or notification that tells you when it takes or releases focus.

I needed to see the full sequence of responder transitions across the entire app, including those private views.

## The swizzle

I added an extension on `UIResponder`:

```swift
extension UIResponder {

    static func swizzleBecomeFirstResponder() {
        let originalBecome = class_getInstanceMethod(UIResponder.self, #selector(becomeFirstResponder))
        let swizzledBecome = class_getInstanceMethod(UIResponder.self, #selector(swizzled_becomeFirstResponder))
        if let original = originalBecome, let swizzled = swizzledBecome {
            method_exchangeImplementations(original, swizzled)
        }
    }

    @objc func swizzled_becomeFirstResponder() -> Bool {
        print("---> becomeFirstResponder: \(type(of: self))")
        return swizzled_becomeFirstResponder()
    }
}
```

The key detail is the recursive-looking call inside `swizzled_becomeFirstResponder`. After `method_exchangeImplementations` runs, the selector `swizzled_becomeFirstResponder` points to the *original* `becomeFirstResponder` implementation. So calling `swizzled_becomeFirstResponder()` from within the replacement is not infinite recursion. It is the call to the original method. This is the standard swizzling pattern.

I activated it in `AppDelegate`, before the window is shown:

```swift
func application(_ application: UIApplication,
                 didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    UIResponder.swizzleBecomeFirstResponder()
    return true
}
```

I added an equivalent swizzle for `resignFirstResponder` to get the full picture.

## What the output revealed

With the swizzle in place, the console printed every responder transition in order:

```
---> becomeFirstResponder: RootNavigationController
---> becomeFirstResponder: BrowserWindow
---> becomeFirstResponder: LaunchScreenViewController
---> resignFirstResponder: LocationView
---> resignFirstResponder: LocationView
---> becomeFirstResponder: BrowserViewController
---> becomeFirstResponder: WKContentView
```

This immediately gave me a complete picture of the hierarchy and the sequence of events, something that would have been impossible to reconstruct from breakpoints alone.

One thing this technique is particularly useful for with `WKWebView` is confirming whether `WKContentView` is becoming first responder or not. `WKContentView` is a private class, so you cannot reference it directly in code or set a targeted breakpoint on it. But because the swizzle is on `UIResponder`, the base class for the entire responder chain, it catches everything, including private UIKit internals. If `WKContentView` takes focus, it shows up in the log like any other class.

Without the swizzle, none of this sequence was visible. The log turned an opaque focus problem into a readable timeline.

## When to reach for this

Swizzling is not a tool you leave in production code. It changes global behaviour for every instance of a class, in every part of the app, for the lifetime of the process. That is too broad for anything except a debugging session.

The right workflow:

1. Add the swizzle behind a `#if DEBUG` block, or simply in a debug branch
2. Reproduce the bug and read the log
3. Fix the root cause
4. Remove the swizzle entirely before shipping

There is also a correctness requirement: swizzling must happen exactly once per method. Calling `method_exchangeImplementations` twice on the same pair restores the original state, effectively a no-op. If you call it from multiple places or inside code that runs more than once, you will get unpredictable results. Using `AppDelegate.application(_:didFinishLaunchingWithOptions:)` is a safe call site because it runs exactly once.

## What to reach for first

Before swizzling, check the cheaper options:

- `UIApplication.shared.keyWindow?.perform(#selector(UIResponder.resignFirstResponder))` to manually clear focus
- A recursive walk of the view hierarchy that checks `isFirstResponder`
- Breakpoint on `becomeFirstResponder` in Xcode's symbolic breakpoint list.

Swizzling is the right choice when the problem spans multiple classes, involves a sequence of events, or is too fast to catch with a debugger. For a first responder bug that crosses view controllers and windows, it was the right call.
