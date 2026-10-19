---
title: Functional Core, Imperative Shell
paths:
  - "src/**/*"
impact: HIGH
---

# Functional Core, Imperative Shell

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Keep business logic in pure functions that take inputs and return outputs with no side effects. Push all side effects -- database calls, HTTP requests, logging, file I/O, and state mutations -- to an outer "imperative shell" that orchestrates the pure core. Pure functions are deterministic: given the same inputs they always produce the same outputs. This makes them trivially testable without mocks, easy to reason about, and safe to compose and parallelize. When side effects are mixed into calculation logic, tests become slow and brittle (requiring database stubs, log spies, HTTP interceptors), bugs hide behind non-deterministic execution, and refactoring becomes dangerous because any change might alter when and how I/O occurs. Separate what to compute from how to execute it.

## Incorrect

Business calculation is tangled with logging, database reads, and persistence. Testing the pricing logic requires mocking the logger, database, and notification service.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

suspend fun applySubscriptionRenewal(
    customerId: String,
    logger: Logger,
    database: RenewalDatabase,
    mailer: Mailer,
) {
    val customer = database.findCustomer(customerId)
    val plan = database.findPlan(customer.planId)

    // Pure calculation mixed with side effects; amounts are in USD.
    var price = plan.basePrice
    if (customer.loyaltyYears >= 3) {
        price = (price * BigDecimal("0.85")).setScale(2, RoundingMode.HALF_UP)
        logger.info("Applied 15% loyalty discount for $customerId")
    }
    if (customer.referralCount >= 5) {
        price -= BigDecimal("10.00")
        logger.info("Applied USD 10 referral credit for $customerId")
    }
    val tax = (price * customer.taxRate).setScale(2, RoundingMode.HALF_UP)
    val total = price + tax

    database.createInvoice(Invoice(customerId, total, tax))
    mailer.send(customer.email, "Your renewal total is USD $total")
    logger.info("Renewal processed: $customerId, total: $total")
}
```

## Correct

The pure core calculates the renewal price with no side effects. The imperative shell fetches data, calls the pure function, then performs all I/O. The core is testable with plain assertions and zero mocks.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

// Pure core — deterministic, no side effects, trivially testable
// Monetary amounts are in USD; taxRate is a decimal fraction (for example, 0.20).
data class RenewalInput(
    val basePrice: BigDecimal,
    val loyaltyYears: Int,
    val referralCount: Int,
    val taxRate: BigDecimal,
)

data class RenewalResult(
    val price: BigDecimal,
    val tax: BigDecimal,
    val total: BigDecimal,
    val appliedDiscounts: List<String>,
)

fun calculateRenewal(input: RenewalInput): RenewalResult {
    val discounts = mutableListOf<String>()
    var price = input.basePrice

    if (input.loyaltyYears >= 3) {
        price = (price * BigDecimal("0.85")).setScale(2, RoundingMode.HALF_UP)
        discounts += "loyalty_15pct"
    }
    if (input.referralCount >= 5) {
        price -= BigDecimal("10.00")
        discounts += "referral_credit_10"
    }

    val tax = (price * input.taxRate).setScale(2, RoundingMode.HALF_UP)
    return RenewalResult(price, tax, price + tax, discounts.toList())
}

// Imperative shell — orchestrates I/O around the pure core
suspend fun processRenewal(
    customerId: String,
    database: RenewalDatabase,
    mailer: Mailer,
    logger: Logger,
) {
    val customer = database.findCustomer(customerId)
    val plan = database.findPlan(customer.planId)
    val result = calculateRenewal(
        RenewalInput(
            basePrice = plan.basePrice,
            loyaltyYears = customer.loyaltyYears,
            referralCount = customer.referralCount,
            taxRate = customer.taxRate,
        )
    )

    database.createInvoice(Invoice(customerId, result.total, result.tax))
    mailer.send(customer.email, "Your renewal total is USD ${result.total}")
    logger.info("Renewal processed: $customerId, result: $result")
}
```

## Reference

- [Functional Core, Imperative Shell - Gary Bernhardt](https://www.destroyallsoftware.com/screencasts/catalog/functional-core-imperative-shell)
- [Boundaries - Gary Bernhardt (talk)](https://www.destroyallsoftware.com/talks/boundaries)
