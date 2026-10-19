---
title: Explicit Data Flow
paths:
  - "src/**/*"
impact: HIGH
---

# Explicit Data Flow

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

If a function produces a result, return it. Never rely on mutation of an input parameter to communicate output. Data should flow explicitly through return values so the reader can trace where each value comes from. When a function mutates its input, the data flow is hidden -- you cannot tell from the call site what changed or whether other references were affected. With explicit returns and `val` bindings to immutable data, the call site becomes self-documenting: assignments show data origin, and immutability prevents downstream code from silently altering what you are reading. A `val` prevents reassignment of the reference; it does not make the referenced object immutable. Kotlin's read-only `Map` interface also does not prevent another reference from mutating its backing data, so pure transformations must create new values without exposing mutable aliases. Prefer pure expressions that produce new values over procedures that modify existing ones.

## Incorrect

Mutation of the input hides where data ends up. The call site looks like a standalone command, but the caller depends on the side effect for its result.

```kotlin
val result = mutableMapOf<String, Boolean>()
if (featureEnabled) {
    applyNewFeature(result) // mutates result in place
}
```

Using `var` and reassignment is still unclear because the reader cannot tell whether `applyNewFeature` also mutates the original in addition to returning a value.

```kotlin
var result = mutableMapOf<String, Boolean>()
if (featureEnabled) {
    result = applyNewFeature(result) // unclear whether applyNewFeature also mutates
}
```

## Correct

A pure expression with `val` makes data flow visible in one line. Here, `baseData` and the result use `Map<String, Boolean>`, and `applyNewFeature` returns a new map without mutating its input. The reader sees exactly what `result` is bound to, with no hidden mutation.

```kotlin
val result = if (featureEnabled) applyNewFeature(baseData) else emptyMap()
```

## Reference

- [Referential Transparency](https://en.wikipedia.org/wiki/Referential_transparency)
- [Immutability and Pure Functions](https://mostly-adequate.gitbook.io/mostly-adequate-guide/ch03)
