---
title: Explicit Side Effects and Call-Site Transparency
paths:
  - "src/**/*"
impact: HIGH
---

# Explicit Side Effects and Call-Site Transparency

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

A reader must understand what a line of code does without opening the called function. When `processOrder(order)` internally saves to a database, sends an email, and emits an event, the call site is opaque — the reader must jump into the implementation to learn what actually happens. This defeats abstraction because it hides critical information rather than irrelevant detail.

Make side effects visible where they are triggered. Each step in a workflow — persistence, notifications, external calls — should appear as a distinct line at the call site. The orchestrating function becomes a transparent table of contents: a reader sees every effect the system produces without drilling into any implementation.

This rule governs call-site composition, not individual function design. Individual functions should still be focused and well-named (see Principle of Least Astonishment). This rule ensures the orchestration of those functions is itself readable.

`CheckoutRequest`, `CheckoutResponse`, and `OrderCompleted` are application types; repository, gateway, notification, and event ports expose suspending operations. These examples are independent of an HTTP framework.

## Incorrect

The call site delegates everything to a single opaque function. A reader sees one line and has no idea that it persists data, charges a payment, sends a confirmation email, and publishes a domain event. The only way to learn this is to open `processOrder`.

```kotlin
// OrderController.kt — opaque orchestration
suspend fun handleCheckout(request: CheckoutRequest): CheckoutResponse {
    val order = buildOrder(request.body)

    // What does this do? Saves to DB? Sends email? Charges payment?
    // Reader must open processOrder() to find out.
    processOrder(order)

    return CheckoutResponse(status = "ok")
}

// OrderService.kt — all side effects hidden inside
suspend fun processOrder(order: Order) {
    orderRepository.save(order)
    paymentGateway.charge(order.customerId, order.total)
    emailService.sendConfirmation(order.customerId, order.id)
    eventBus.publish(OrderCompleted(orderId = order.id))
}
```

## Correct

Every side effect is visible at the call site. A reader scanning `handleCheckout` sees exactly what the system does — persist, charge, notify, publish — without opening any implementation. Each called function does one focused thing, and the orchestrator makes the workflow transparent.

```kotlin
// OrderController.kt — transparent orchestration
suspend fun handleCheckout(request: CheckoutRequest): CheckoutResponse {
    val order = buildOrder(request.body)

    // Every effect is visible right here
    orderRepository.save(order)
    paymentGateway.charge(order.customerId, order.total)
    emailService.sendConfirmation(order.customerId, order.id)
    eventBus.publish(OrderCompleted(orderId = order.id))

    return CheckoutResponse(status = "ok")
}
```
