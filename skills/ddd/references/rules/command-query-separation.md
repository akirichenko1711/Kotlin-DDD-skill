---
title: Command-Query Separation (CQS)
paths:
  - "src/**/*"
impact: HIGH
---

# Command-Query Separation (CQS)

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

A function must either return a value (query) or cause a side effect (command), never both. Mixing the two makes call sites deceptive: a mutation disguised as a query hides state changes, and a query that secretly throws hides control flow. Separate queries from commands so that assignments signal pure data retrieval and standalone calls signal state changes. When you need both a result and a side effect, split the operation into two explicit steps.

## Incorrect Mutation

`applyNewFeature(result)` mutates its input but the caller uses the mutated object as if it were a return value. The mutation is invisible at the call site.

```kotlin
val result = mutableMapOf<String, Boolean>()
if (featureEnabled) {
    applyNewFeature(result) // mutates result — output is hidden in the input
}
```

Reassignment does not fix it when the function both mutates AND returns. The caller cannot tell whether the original was changed.

```kotlin
var result = mutableMapOf<String, Boolean>()
if (featureEnabled) {
    result = applyNewFeature(result) // unclear: does it mutate AND return?
}
```

## Correct Pure Function

Pure expression that returns a new value without mutating input. The call site clearly shows this is a query.

```kotlin
val result = if (featureEnabled) applyNewFeature(baseData) else emptyMap()
```

## Incorrect Hidden Command

`validateResult` looks like a query but secretly throws, making it a hidden command. The call site hides a control flow branch.

```kotlin
val result = performProcess(param)
validateResult(result) // throws — the call site hides the failure branch
```

## Correct Explicit Control Flow

Explicit control flow at the call site. The caller decides what to do with an invalid result instead of a hidden throw.

```kotlin
val result = performProcess(param)
if (!isValid(result)) {
    throw ProcessingError(result)
}
```

## Reference

- [CommandQuerySeparation by Martin Fowler](https://martinfowler.com/bliki/CommandQuerySeparation.html)
- [Command-Query Separation by Bertrand Meyer](https://en.wikipedia.org/wiki/Command%E2%80%93query_separation)
