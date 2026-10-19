---
title: Avoid Code Duplication — Function, Logic, Concept, and Pattern
paths:
  - "src/**/*"
impact: HIGH
---

# Avoid Code Duplication — Function, Logic, Concept, and Pattern

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

- Do NOT duplicate functions, business logic, domain concepts, or behavioral patterns. 
- Apply DRY (Hunt & Thomas): "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system." 
- Allways extract on the third occurrence (Fowler's Rule of Three).

## Incorrect — Function Duplication

Identical bodies copy-pasted across modules. The repository examples use an application-owned `DocumentCollection<T>` port with a suspending `findById` operation.

```kotlin
// UserRepository.kt
suspend fun findUserById(id: String): User? = userCollection.findById(id)

// ProductRepository.kt — identical body, different name
suspend fun findProductById(id: String): Product? = productCollection.findById(id)
```

## Correct — Function Duplication

Extract a generic function; callers specify only what differs.

```kotlin
// Repository.kt
suspend fun <T> findById(collection: DocumentCollection<T>, id: String): T? =
    collection.findById(id)

suspend fun findUserById(id: String): User? = findById(userCollection, id)
suspend fun findProductById(id: String): Product? = findById(productCollection, id)
```

## Incorrect — Logic Duplication

Same business rule in three services with different variable names. The arithmetic uses `Double` for brevity; use the money representation chosen by the domain for real pricing. More subtle than function duplication — code looks different but encodes the same decision. When thresholds change, missed sites silently drift.

```kotlin
// OrderService.kt
fun calculateOrderDiscount(order: Order): Double = when {
    order.total > 500.0 -> order.total * 0.1
    order.total > 200.0 -> order.total * 0.05
    else -> 0.0
}

// InvoiceService.kt — same rule, different names and types
fun getInvoiceDiscount(invoice: Invoice): Double = when {
    invoice.amount > 500.0 -> invoice.amount * 0.1
    invoice.amount > 200.0 -> invoice.amount * 0.05
    else -> 0.0
}

// ReportService.kt — same thresholds embedded in a sum
fun getDiscountedRevenue(transactions: List<Transaction>): Double =
    transactions.sumOf { transaction ->
        val discount = when {
            transaction.amount > 500.0 -> 0.1
            transaction.amount > 200.0 -> 0.05
            else -> 0.0
        }
        transaction.amount * (1 - discount)
    }
```

## Correct — Logic Duplication

One domain function owns the rule. Changing thresholds happens in exactly one place.

```kotlin
// Pricing.kt — single source of truth
fun getDiscountRate(amount: Double): Double = when {
    amount > 500.0 -> 0.1
    amount > 200.0 -> 0.05
    else -> 0.0
}

// OrderService.kt
val orderDiscount = order.total * getDiscountRate(order.total)

// InvoiceService.kt
val invoiceDiscount = invoice.amount * getDiscountRate(invoice.amount)

// ReportService.kt
val revenue = transactions.sumOf { transaction ->
    transaction.amount * (1 - getDiscountRate(transaction.amount))
}
```

## Incorrect — Concept Duplication

The concept "active user" is scattered as ad-hoc conditions across modules. Most dangerous form — code differs so tools will not flag it, yet every instance must stay in sync. Missed sites become silent bugs.

```kotlin
// AuthMiddleware.kt
if (user.status == UserStatus.ACTIVE && user.deletedAt == null && user.emailVerified) {
    allowAccess(user)
}

// NotificationService.kt — subtly different expression
if (user.status == UserStatus.ACTIVE && user.deletedAt == null && user.emailVerified == true) {
    sendNotification(user)
}

// BillingService.kt — concept drift: forgot emailVerified
if (user.status == UserStatus.ACTIVE && user.deletedAt == null) {
    chargeSubscription(user)
}

// AnalyticsService.kt — further drift: added its own interpretation
if (user.status == UserStatus.ACTIVE && user.deletedAt == null && user.lastLoginAt != null) {
    trackActiveUser(user)
}
```

## Correct — Concept Duplication

Name the concept in a single predicate. When requirements change, update one function.

```kotlin
// UserStatus.kt — authoritative definition
fun isActiveUser(user: User): Boolean =
    user.status == UserStatus.ACTIVE && user.deletedAt == null && user.emailVerified

// AuthMiddleware.kt
if (isActiveUser(user)) allowAccess(user)

// NotificationService.kt
if (isActiveUser(user)) sendNotification(user)

// BillingService.kt — now correct
if (isActiveUser(user)) chargeSubscription(user)

// AnalyticsService.kt — shared definition + own criteria
if (isActiveUser(user) && user.lastLoginAt != null) trackActiveUser(user)
```

## Incorrect — Pattern Duplication

Same fetch-validate-transform pattern repeated per API resource. Here `api` is an application-owned HTTP port whose suspending `get` returns a typed response with `status` and `body`; the resource decoders are also application functions.

```kotlin
import java.time.Instant

// UserApi.kt
suspend fun fetchUser(id: String): Fetched<User> {
    val response = api.get("/api/users/$id")
    if (response.status !in 200..299) throw ApiException("Failed: ${response.status}")
    return Fetched(decodeUser(response.body), Instant.now())
}

// ProductApi.kt — same pattern, different resource
suspend fun fetchProduct(id: String): Fetched<Product> {
    val response = api.get("/api/products/$id")
    if (response.status !in 200..299) throw ApiException("Failed: ${response.status}")
    return Fetched(decodeProduct(response.body), Instant.now())
}
```

## Correct — Pattern Duplication

Extract the recurring pattern into a generic abstraction.

```kotlin
import java.time.Instant

// ApiClient.kt
// A wrapper adds metadata without pretending it is part of every resource type.
data class Fetched<T>(val value: T, val fetchedAt: Instant)

suspend fun <T> fetchResource(
    resource: String,
    id: String,
    decode: (String) -> T,
): Fetched<T> {
    val response = api.get("/api/$resource/$id")
    if (response.status !in 200..299) throw ApiException("Failed: ${response.status}")
    return Fetched(decode(response.body), Instant.now())
}

suspend fun loadResources(id: String): Pair<Fetched<User>, Fetched<Product>> {
    val user = fetchResource("users", id, ::decodeUser)
    val product = fetchResource("products", id, ::decodeProduct)
    return user to product
}
```

## Reference

- [The Pragmatic Programmer — Hunt & Thomas](https://pragprog.com/titles/tpp20/the-pragmatic-programmer-20th-anniversary-edition/) — DRY principle
- [Refactoring — Martin Fowler](https://refactoring.com/) — Rule of Three
- [Extreme Programming Explained — Kent Beck](https://www.oreilly.com/library/view/extreme-programming-explained/0201616416/) — Once and Only Once (OAOO)
