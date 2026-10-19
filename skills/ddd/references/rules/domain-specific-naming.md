---
title: Use Domain-Specific Names Instead of Generic Module Names
paths:
  - "**/*"
  - "**/*"
impact: HIGH
---

# Use Domain-Specific Names Instead of Generic Module Names

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Avoid generic module names like `utils`, `helpers`, `common`, and `shared`. These names attract unrelated functions, creating grab-bag files with no cohesion. Use domain-specific names that reflect the bounded context and the module's single responsibility -- names like `OrderCalculator`, `UserAuthenticator`, or `InvoiceGenerator` make purpose immediately clear and enforce cohesion by design.

Generic names signal missing domain analysis. When a developer reaches for `Utils.kt`, it usually means the function belongs in a domain module that has not been identified yet. Naming modules after their domain concept prevents them from becoming dumping grounds and keeps each module focused on a single, clear purpose.

## Critical princeples

- Follow domain-driven design and ubiquitous language
- **AVOID** generic names: `utils`, `helpers`, `common`, `shared`
- **USE** domain-specific names: `OrderCalculator`, `UserAuthenticator`, `InvoiceGenerator`
- Follow bounded context naming patterns
- Each module should have a single, clear purpose

## Incorrect

Generic module names attract unrelated functions, making the file a dumping ground with no cohesion or clear ownership.

```kotlin
import java.math.BigDecimal

// Utils.kt — grab-bag of unrelated functions
fun calculateOrderTotal(items: List<OrderItem>): BigDecimal =
    items.fold(BigDecimal.ZERO) { sum, item ->
        sum + item.price * item.quantity.toBigDecimal()
    }

fun formatUserDisplayName(user: User): String =
    "${user.firstName} ${user.lastName}"

fun generateInvoiceNumber(): String = "INV-${System.currentTimeMillis()}"
```

Generic Naming Anti-Patterns:
- `Utils.kt` with 50 unrelated functions
- `helpers/Misc.kt` as a dumping ground
- `common/Shared.kt` with unclear purpose

## Correct

Each function lives in a module named after its bounded context, enforcing single responsibility and making purpose self-documenting.

```kotlin
import java.math.BigDecimal

// OrderCalculator.kt — all order pricing logic
fun calculateOrderTotal(items: List<OrderItem>): BigDecimal =
    items.fold(BigDecimal.ZERO) { sum, item ->
        sum + item.price * item.quantity.toBigDecimal()
    }

// UserDisplay.kt — user presentation formatting
fun formatUserDisplayName(user: User): String =
    "${user.firstName} ${user.lastName}"

// InvoiceGenerator.kt — invoice creation logic
fun generateInvoiceNumber(): String = "INV-${System.currentTimeMillis()}"
```

## Reference

- [Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.domainlanguage.com/ddd/) — Eric Evans
- Source: `plugins/ddd/skills/software-architecture/SKILL.md` — Naming Conventions and Generic Naming Anti-Patterns
