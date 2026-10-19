---
title: Principle of Least Astonishment — Functions Must Do Only What Their Name Promises
paths:
  - "src/**/*"
impact: HIGH
---

# Principle of Least Astonishment — Functions Must Do Only What Their Name Promises

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

A function must do exactly what its name and signature suggest — nothing more, nothing less. Hidden side effects violate caller expectations, create invisible coupling, and make code unpredictable. When a function named `getUser` also emits analytics events, or a function named `validate` throws instead of returning a boolean, callers cannot reason about behavior from the interface alone. This forces every developer to read the implementation before using the function, defeating the purpose of abstraction.

Keep functions honest: if a name implies a pure query, do not mutate state. If a name implies validation, return a result rather than throwing. Make all side effects explicit at the call site so the reader's mental model matches the actual execution. When additional work is needed (logging, analytics, notifications), perform it in the calling code or use clearly named wrapper functions that advertise the combined behavior.

## Incorrect

The function name `getUser` promises a data retrieval operation, but it secretly logs an analytics event and updates a last-accessed timestamp — side effects the caller never asked for and cannot see from the signature.

```kotlin
import java.time.Instant

// UserService.kt — hidden side effects inside a getter
suspend fun getUser(userId: String): User {
    val user = userRepository.findById(userId)
        ?: throw NoSuchElementException("User not found")

    // Hidden side effect: analytics tracking
    analyticsService.track(
        "user_viewed",
        mapOf("userId" to user.id, "timestamp" to Instant.now().toString()),
    )

    // Hidden side effect: mutates database state
    userRepository.updateLastAccessed(userId, Instant.now())

    return user
}
```

## Correct

The getter does only what its name says — retrieves a user. Side effects are performed explicitly at the call site, where the caller can see and control them. The call-site fragment after the getter belongs inside a suspending application function.

```kotlin
import java.time.Instant

// UserService.kt — getter does only what the name promises
suspend fun getUser(userId: String): User =
    userRepository.findById(userId)
        ?: throw NoSuchElementException("User not found")

// call site — side effects are explicit and visible
val user = getUser(userId)
analyticsService.track("user_viewed", mapOf("userId" to user.id))
userRepository.updateLastAccessed(userId, Instant.now())
```

## Reference

- [Principle of Least Astonishment (Wikipedia)](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)
- [Clean Code by Robert C. Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) — Chapter 3: Functions should do one thing
