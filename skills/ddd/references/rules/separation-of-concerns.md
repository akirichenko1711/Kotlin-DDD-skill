---
title: Enforce Separation of Concerns Between Layers
paths:
  - "src/**/*"
impact: HIGH
---

# Enforce Separation of Concerns Between Layers

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).
> Revised 2026-10-19: align the example with the shared Order aggregate, pricing policy, and creation use case.

Do NOT mix business logic with UI components or place database queries directly in controllers. Each layer has a distinct responsibility: controllers handle HTTP concerns, application use cases coordinate work, the domain model owns business rules and valid aggregate creation, and infrastructure adapters handle storage. Moving calculations from a controller into an application service is not enough when it leaves the domain model unable to protect its own rules. Maintain clear boundaries so the same domain behavior can be used from an API, CLI, or event handler.

## Critical principles

- Do NOT mix business logic with UI components
- Keep database queries out of controllers
- Keep pricing rules and aggregate creation constraints in the domain model
- Let application use cases coordinate domain operations and persistence
- Maintain clear boundaries between contexts
- Ensure proper separation of responsibilities

This example uses the same `Order`, `OrderItem`, `OrderPricing`, and `OrderRepository` defined in [Separate Domain Logic from Infrastructure](clean-architecture-ddd.md#correct). The fixed policy is 10% off orders above 100.00. Domain definitions are kept there rather than duplicated here. As in that example, amounts use one currency and input items are already priced; catalog lookup, transport mapping, and database mapping are omitted.

## Incorrect

The controller calculates the price, writes raw data directly, and constructs the HTTP response. It bypasses the aggregate's creation path, so checks such as rejecting an empty order do not run. `OrderDatabase` is an application-owned database adapter; request, response, and returned database-record types are illustrative application types.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

// OrderController.kt — everything in one place
class OrderController(private val database: OrderDatabase) {
    suspend fun createOrder(request: CreateOrderRequest): OrderResponse {
        // Business logic mixed into the controller
        val subtotal = request.items.fold(BigDecimal.ZERO) { sum, item ->
            sum + item.price * item.quantity.toBigDecimal()
        }
        val total = if (subtotal > BigDecimal("100.00")) {
            (subtotal * BigDecimal("0.90")).setScale(2, RoundingMode.HALF_UP)
        } else {
            subtotal
        }

        // Direct database access instead of creating and saving an aggregate
        val order = database.insertOrder(request.customerId, total, request.items)
        return OrderResponse(status = 201, body = order)
    }
}
```

## Correct

The same creation path as the domain-boundary example is now split by responsibility:

- `OrderController` delegates the HTTP request to `CreateOrder`.
- `CreateOrder` asks `Order.create` for a complete valid aggregate, then saves it through `OrderRepository`. It contains no discount calculation.
- The shared `Order` and `OrderPricing` own the creation constraints and price calculation.
- `SqlOrderRepository` implements the storage contract. `saveAggregate` represents the database adapter's atomic save of the root and its items; the database transaction and mapping implementation are omitted.

```kotlin
import java.util.UUID

// Reuse the domain Order, OrderItem, OrderPricing, and OrderRepository
// from the linked example; their definitions are not repeated here.

// OrderController.kt — handles HTTP only
class OrderController(private val createOrder: CreateOrder) {
    suspend fun createOrder(request: CreateOrderRequest): OrderResponse {
        val order = createOrder.execute(request.customerId, request.items)
        return OrderResponse(status = 201, body = order)
    }
}

// CreateOrder.kt — application orchestration, identical to the linked example
class CreateOrder(private val orders: OrderRepository) {
    suspend fun execute(customerId: String, items: List<OrderItem>): Order {
        val order = Order.create(UUID.randomUUID(), customerId, items)
        orders.save(order)
        return order
    }
}

// SqlOrderRepository.kt — data access only
class SqlOrderRepository(private val database: OrderDatabase) : OrderRepository {
    override suspend fun save(order: Order) {
        database.saveAggregate(order)
    }
}
```

## Reference

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
