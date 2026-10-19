---
title: Use Existing Libraries Instead of Custom Code
impact: HIGH
paths:
  - "**/*"
  - "**/*"
---

# Use Existing Libraries Instead of Custom Code

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Always search for existing libraries, services, or third-party APIs before writing custom code:
- Check Maven Central and the Kotlin ecosystem for existing libraries that solve the problem
- Evaluate existing services/SaaS solutions
- Consider third-party APIs for common functionality

Every line of custom code is a liability that requires maintenance, testing, and documentation. The Not Invented Here (NIH) syndrome leads to fragile, undertested reimplementations of solved problems.

Before writing any utility, helper, or infrastructure code, check Maven Central and the Kotlin ecosystem for established libraries that solve the problem. For example, use Resilience4j instead of writing your own retry logic.

Custom code is only justified when:
- Specific business logic unique to the domain
- Performance-critical paths with special requirements
- When external dependencies would be overkill
- Security-sensitive code requiring full control
- When existing solutions don't meet requirements after thorough evaluation

## Incorrect

Custom retry logic is implemented from scratch instead of using an established library. This hand-rolled solution lacks features like exponential backoff, jitter, circuit breaking, and proper error classification that battle-tested libraries provide.

```kotlin
import kotlinx.coroutines.delay

// Custom retry utility — reinventing the wheel
suspend fun <T> retry(
    maxAttempts: Int = 3,
    delayMillis: Long = 1_000,
    operation: suspend () -> T,
): T {
    require(maxAttempts > 0)
    var lastError: Exception? = null
    repeat(maxAttempts) {
        try {
            return operation()
        } catch (error: Exception) {
            lastError = error
            delay(delayMillis)
        }
    }
    throw checkNotNull(lastError)
}

suspend fun loadUsers(fetchFromApi: suspend (String) -> List<User>): List<User> =
    retry { fetchFromApi("/users") }
```

Anti-Pattern to Avoid: NIH (Not Invented Here) Syndrome:
- Don't build custom auth when Auth0/Supabase exists
- Don't write custom state management instead of using established Kotlin libraries
- Don't create custom form validation instead of using established libraries

## Correct

An established library provides retry policies with backoff, jitter, and error classification, and can be composed with a circuit breaker when needed. This example uses the `resilience4j-kotlin` and `resilience4j-retry` modules; only I/O failures are retried, so coroutine cancellation and business errors propagate immediately.

```kotlin
import io.github.resilience4j.core.IntervalFunction
import io.github.resilience4j.kotlin.retry.executeSuspendFunction
import io.github.resilience4j.retry.Retry
import io.github.resilience4j.retry.RetryConfig
import java.io.IOException

val retryPolicy: Retry = Retry.of(
    "users-api",
    RetryConfig.custom<Any>()
        .maxAttempts(3) // Includes the initial attempt.
        .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(1_000L, 2.0, 0.5))
        .retryOnException { it is IOException }
        .build(),
)

suspend fun loadUsers(fetchFromApi: suspend (String) -> List<User>): List<User> =
    retryPolicy.executeSuspendFunction { fetchFromApi("/users") }
```

## Reference

- [Resilience4j Kotlin support](https://resilience4j.readme.io/docs/getting-started-4)
- [Resilience4j retry configuration](https://resilience4j.readme.io/docs/retry)
