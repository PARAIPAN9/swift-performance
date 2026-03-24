---
layout: post
title: "Copy-on-Write: It's Not About Struct vs Class"
description: "Most Swift performance articles frame CoW as a struct optimization. It's not. It's a semantics decision and the benchmark data shows exactly when it pays off."
categories: [ARC, Value/Reference Types, Copy-on-Write]
---

A common misconception in Swift performance discussions is that copy-on-write is a way to make structs behave like classes for performance. It isn't. CoW is a **semantics decision**. You're choosing when storage is duplicated, not choosing between value and reference semantics.

This article walks through a concrete implementation, explains what happens at the ARC level, and backs every claim with benchmark data.

## The problem with heap-heavy structs

Consider a struct with 10 fields, all heap-allocated:

```swift
public struct HeavyStruct {
    public var name: String           // heap buffer + ARC
    public var description: String
    public var identifier: String
    public var category: String
    public var tags: [String]         // heap array + ARC retain per element
    public var keywords: [String]
    public var authors: [String]
    public var relatedIDs: [String]
    public var scores: [Int]          // heap array
    public var metadata: [Int]
}
```

When you copy this struct, Swift retains every reference individually. That's roughly **10+ ARC retain calls per copy**. One for each `String`, one for each `Array` buffer.

The retains are cheap individually, but they add up.

## What CoW actually does

Copy-on-Write moves all fields into a single class instance (`_Storage`) and wraps it in the struct. Basically, it is a struct, backed by a class:

```swift
public struct HeavyCOWStruct {

    final class Storage {
        var name: String
        var description: String
        var identifier: String
        var category: String
        var tags: [String]
        var keywords: [String]
        var authors: [String]
        var relatedIDs: [String]
        var scores: [Int]
        var metadata: [Int]

        func copy() -> Storage { ... }
    }

    private var _storage: Storage
}
```

Now copying the struct is **1 ARC retain** on `_storage`, regardless of how many fields it contains. The actual buffer duplication is deferred to the first mutation and only if another owner of the same storage exists:

```swift
public var name: String {
    get { _storage.name }
    set {
        // Only allocates if someone else holds a reference to _storage.
        if !isKnownUniquelyReferenced(&_storage) {
            _storage = _storage.copy()
        }
        _storage.name = newValue
    }
}
```

`isKnownUniquelyReferenced` is an O(1) check on the ARC reference count. If the count is 1, meaning no other variable holds this storage, the mutation happens in place with zero allocation.

## This is about semantics, not struct vs class

The struct still has **value semantics**. Two variables holding the same `HeavyCOWStruct` are independent values. Mutating one does not affect the other:

```swift
var a = HeavyCOWStruct(name: "Swift", ...)
let b = a                 // b._storage === a._storage — shared, 1 ARC retain
a.name = "Performance"    // new Storage only if b is still alive
// a.name == "Performance", b.name == "Swift"
```

If you used a plain `class` instead, `b` would reflect the mutation. CoW gives you the copy safety of a struct with the copy efficiency of a class, but it is **not free**. Every mutating property setter pays the `isKnownUniquelyReferenced` check, and when storage is shared, it pays a full allocation.

## The benchmark

To measure the difference, I ran two tests, a copy followed by a mutation on both versions. Each test was run with `-c release` to get optimized builds.

**Results:**

| Version | Time per operation |
|---|---|
| `HeavyStruct` (plain) | 7.42 µs |
| `HeavyCOWStruct` (CoW) | 5.40 µs |

**CoW was ~27% faster.**

The mutation makes the difference clear. The plain struct is fully copied at assignment (all 10 ARC retains), then mutated in place. The CoW struct pays only 1 ARC retain at assignment. The mutation checks `isKnownUniquelyReferenced` and because `original` is no longer in use at that point, the reference count is 1, so no `Storage.copy()` is needed. The mutation happens in place.

CoW wins both steps: cheaper copy, free mutation.

## When CoW does not help

CoW is not always the right choice. It adds overhead when:

**Mutations are frequent with shared owners.** If two variables hold the same CoW struct and you mutate through one of them, `Storage.copy()` runs a full heap allocation of all fields. This is more expensive than a plain struct mutation.

**Fields are small scalars.** A struct with `Bool`, `Int`, and `Float` fields has nothing to gain. There are no heap references to consolidate. The struct copies by value directly.

**Instances are always freshly constructed.** CoW shares storage between copies of the *same instance*. If every use of the struct calls `init`, there is never a shared storage to benefit from.

**Property access is in a tight loop.** Every read goes through `_storage.field` an extra pointer indirection compared to a plain struct. For a type accessed thousands of times per frame, this adds up.

## Advice

> Use CoW when a struct has many reference-type fields, is frequently copied, and mutations either don't happen or happen while the copy is the sole owner. Skip it for small structs, scalar-heavy types, or anything mutated in a tight loop with shared owners.

The data, not the pattern should drive the decision.

## Further reading

- [CoW in practice: swift-nio-examples](https://github.com/apple/swift-nio-examples/blob/main/http2-client/Sources/http2-client/Types.swift#L22): A real-world example from Apple's SwiftNIO team showing how CoW-backed value types are used in production networking code.
- [High-Performance Systems In Swift](https://www.youtube.com/watch?v=iLDldae64xE): Apple engineer, Johannes Weiss from the SwiftNIO team, walks through the performance model for value types, reference types and the gradual steps to a high performant system. Essential context for every decision covered in this article.
