---
title: Typed Error Handling with Logging
paths:
  - "src/**/*"
impact: HIGH
---

# Typed Error Handling with Logging

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Never silently swallow failures. Use typed catch blocks that distinguish between expected domain errors and unexpected system failures, and log failures with sufficient context (operation name, relevant IDs) before rethrowing or returning a failure result. Broad `catch (error: Exception)` blocks that ignore the exception hide the root cause, making production debugging nearly impossible. When wrapping an exception, preserve the original exception as its cause. When error handling logic grows complex, extract it into smaller, reusable handler functions rather than duplicating catch logic across the codebase.

Coroutine cancellation is control flow: rethrow `CancellationException` unchanged before any broad exception handler. Do not turn cancellation into a payment failure or log it as an unexpected system error.

## Incorrect

Errors are caught with a broad generic block and silently swallowed. This also swallows coroutine cancellation. The caller has no trace of what failed or why.

```kotlin
suspend fun processPayment(orderId: String, amount: Long): PaymentReceipt? {
  try {
    val result = paymentGateway.charge(orderId, amount)
    orders.markPaid(orderId)
    return result
  } catch (_: Exception) {
    // Silently swallowed — no logging, no typed handling
    return null
  }
}
```

## Correct

Payment failures are caught by type and logged with context and their original throwable. Known exceptions are wrapped with their cause intact; unexpected exceptions and cancellation propagate unchanged. The logger below is `java.util.logging.Logger`; the gateway, repository, and exception classes are application types.

```kotlin
import java.util.logging.Level
import kotlinx.coroutines.CancellationException

suspend fun processPayment(orderId: String, amount: Long): PaymentReceipt {
  try {
    val result = paymentGateway.charge(orderId, amount)
    orders.markPaid(orderId)
    return result
  } catch (cancelled: CancellationException) {
    throw cancelled
  } catch (error: PaymentDeclinedException) {
    logger.log(
      Level.WARNING,
      "processPayment declined: orderId=$orderId, amount=$amount, reason=${error.reason}",
      error,
    )
    throw DomainException("Payment declined for order $orderId", error)
  } catch (error: NetworkException) {
    logger.log(
      Level.SEVERE,
      "processPayment gateway unreachable: orderId=$orderId, amount=$amount",
      error,
    )
    throw InfrastructureException("Payment service unavailable", error)
  } catch (error: Exception) {
    logger.log(
      Level.SEVERE,
      "Unexpected processPayment failure: orderId=$orderId, amount=$amount",
      error,
    )
    throw error
  }
}
```

## Reference

- [Kotlin — Exception and Error Handling](https://kotlinlang.org/docs/exceptions.html)
- [Kotlin Coroutines — Cancellation and Timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)
