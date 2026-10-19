---
title: Separate Domain Logic from Infrastructure
paths:
  - "src/**/*"
impact: HIGH
---

# Separate Domain Logic from Infrastructure

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Keep business logic in pure domain and use case layers, free of framework or infrastructure dependencies. When domain logic is coupled to controllers, ORMs, or HTTP libraries, it becomes untestable in isolation, impossible to reuse across delivery mechanisms, and fragile to infrastructure changes. Define domain entities that model business rules with no imports from framework or database packages. Implement use cases as classes that depend on abstract repository interfaces, not concrete database clients. Let the infrastructure layer implement those interfaces and inject them at composition time. This dependency inversion ensures the domain drives the architecture rather than the framework dictating how business rules are organized.

## Critical Clean Architecture & DDD Principles

- Separate domain entities from infrastructure concerns
- Keep business logic independent of frameworks
- Define use cases clearly and keep them isolated
- Avoid code duplication through creation of reusable functions and modules

## Incorrect

Business logic is embedded directly in the HTTP adapter and coupled to database access. Testing the pricing rule requires a database or a database stub. `OrderDatabase`, request, and response types represent application-specific infrastructure adapters and transport data; framework setup is omitted.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

// HTTP adapter — business rules and database access mixed together
class OrderController(private val database: OrderDatabase) {
    suspend fun createOrder(request: CreateOrderRequest): OrderResponse {
        val subtotal = request.items.fold(BigDecimal.ZERO) { sum, item ->
            sum + item.price * item.quantity.toBigDecimal()
        }
        val total = if (subtotal > BigDecimal("100.00")) {
            (subtotal * BigDecimal("0.90")).setScale(2, RoundingMode.HALF_UP)
        } else {
            subtotal
        }

        val order = database.insertOrder(request.customerId, total, request.items)
        return OrderResponse(status = 201, body = order)
    }
}
```

Poor Architectural Choices:
- Mixing business logic with UI components
- Database queries directly in controllers
- Lack of clear separation of concerns

## Correct

Domain logic lives in a framework-free use case that depends on an abstract repository. The controller is a thin adapter that delegates to the use case.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

// domain/Order.kt — pure business logic, no framework imports
fun calculateOrderTotal(items: List<OrderItem>): BigDecimal {
    val subtotal = items.fold(BigDecimal.ZERO) { sum, item ->
        sum + item.price * item.quantity.toBigDecimal()
    }
    return if (subtotal > BigDecimal("100.00")) {
        (subtotal * BigDecimal("0.90")).setScale(2, RoundingMode.HALF_UP)
    } else {
        subtotal
    }
}

interface OrderRepository {
    suspend fun save(order: Order): Order
}

// application/CreateOrder.kt — use case depends on an abstraction
class CreateOrder(private val orders: OrderRepository) {
    suspend fun execute(customerId: String, items: List<OrderItem>): Order {
        val order = Order(customerId, calculateOrderTotal(items), items.toList())
        return orders.save(order)
    }
}

// infrastructure/OrderController.kt — thin HTTP adapter
class OrderController(private val createOrder: CreateOrder) {
    suspend fun createOrder(request: CreateOrderRequest): OrderResponse {
        val order = createOrder.execute(request.customerId, request.items)
        return OrderResponse(status = 201, body = order)
    }
}
```

## Reference

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Domain-Driven Design by Eric Evans](https://www.domainlanguage.com/ddd/)
