---
title: Explicit Control Flow and Policy-Mechanism Separation
paths:
  - "src/**/*"
impact: HIGH
---

# Explicit Control Flow and Policy-Mechanism Separation

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Error conditions, branching, and control flow decisions must be visible at the call site — never hidden inside helper functions that look like simple validators or utilities. This is an application of the policy-mechanism separation principle: a "mechanism" is a pure function that computes a result and returns it; a "policy" is what the caller decides to do with that result — throw, log, branch, or ignore.

When policy is hidden inside mechanism (e.g., a `validate` function that throws instead of returning a boolean), the call site becomes deceptive. The reader sees what looks like a passive check but is actually a control flow branch that can halt execution. Keeping mechanisms pure and policies explicit at the call site makes code predictable and composable: the same mechanism can serve different policies without modification.

Apply this separation consistently:

- **Mechanism** = `isValid(result)` returns a boolean. **Policy** = the caller decides to throw.
- **Mechanism** = `applyNewFeature(baseData)` returns new data. **Policy** = the caller decides whether to call it based on a feature flag.
- **Mechanism** = `formatResult(result)` returns a string. **Policy** = the caller decides to log it.

## Incorrect

`validateResult` hides a throw inside what reads like a passive validation check. The call site shows no branching, no `if`, no `throw` — the reader assumes execution continues normally after the call. The control flow decision (throw on invalid) is buried inside the mechanism.

```kotlin
fun validateResult(result: ProcessingResult) {
    if (!result.success) {
        throw ProcessingError(result.error)
    }
    if (result.value < 0) {
        throw IllegalArgumentException("Negative value")
    }
}

// call site — looks harmless, hides two possible throws
val result = performProcess(param)
validateResult(result)
```

Similarly, hiding a feature-flag policy inside the mechanism couples the feature decision to the transformation:

```kotlin
fun applyNewFeature(data: FeatureData): FeatureData {
    if (!featureFlags.isEnabled("new-feature")) {
        return data // policy hidden inside mechanism
    }
    return transform(data)
}

// call site — reader cannot tell a feature flag is being checked
val output = applyNewFeature(baseData)
```

## Correct

The mechanism (`isValid`) is a pure function that returns a value. The policy (what to do when invalid) is explicit at the call site. Every branch point is visible to the reader.

```kotlin
fun isValid(result: ProcessingResult): Boolean =
    result.success && result.value >= 0

// call site — control flow is visible
val result = performProcess(param)
if (!isValid(result)) {
    throw ProcessingError(result)
}
```

The feature-flag policy is at the call site, and the mechanism is a pure transformation:

```kotlin
fun applyNewFeature(data: FeatureData): FeatureData =
    transform(data) // pure mechanism — always transforms

// call site — policy is explicit
val output = if (featureEnabled) applyNewFeature(baseData) else baseData
```

Logging follows the same pattern — the mechanism formats, the caller decides to log:

```kotlin
val summary = formatResult(result) // mechanism: returns a string
logger.info(summary)               // policy: caller decides to log
```

## Reference

- [Policy-Mechanism Separation (Wikipedia)](https://en.wikipedia.org/wiki/Separation_of_mechanism_and_policy)
