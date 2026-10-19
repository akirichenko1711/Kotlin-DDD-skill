---
title: Separate Domain Logic from Infrastructure
paths:
  - "src/**/*"
impact: HIGH
---

# Separate Domain Logic from Infrastructure

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).
> Revised 2026-10-19: make the pricing policy, aggregate boundary, and invariant-preserving creation explicit.

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

The domain owns the pricing policy and valid order creation. The application use case coordinates creation and persistence; it does not assemble a partially valid aggregate or supply a precomputed total. The controller delegates to the use case.

A domain rule does not require a `DomainRule` interface or a separate class: a pure function or an aggregate method can be sufficient. Here, `OrderPricing` names the fixed pricing policy explicitly. `Order.create` is a factory method on the aggregate root; its private constructor is one way to prevent bypassing creation rules. A validating public constructor would also be valid for a simple model, and a separate factory is useful when assembly becomes complex.

In this small model, an order has an identity, at least one item, and a nonblank customer ID. Items have a product ID, a nonnegative price, and a positive quantity. The root owns a snapshot of its immutable items, derives its total, and exposes no way to replace the total independently. These are the example's business assumptions, not universal rules for every ordering domain. Amounts use one currency; catalog pricing and transport/database mapping are outside this example.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode
import java.util.UUID

// domain/OrderItem.kt — immutable value object, not an ORM entity
data class OrderItem(val productId: String, val price: BigDecimal, val quantity: Int) {
    init {
        require(productId.isNotBlank()) { "Product ID is required" }
        require(price.signum() >= 0) { "Price must not be negative" }
        require(quantity > 0) { "Quantity must be positive" }
    }
}

// domain/OrderPricing.kt — the policy: 10% off orders above 100.00
object OrderPricing {
    fun totalFor(items: List<OrderItem>): BigDecimal {
        val subtotal = items.fold(BigDecimal.ZERO) { sum, item ->
            sum + item.price * item.quantity.toBigDecimal()
        }
        return if (subtotal > BigDecimal("100.00")) {
            (subtotal * BigDecimal("0.90")).setScale(2, RoundingMode.HALF_UP)
        } else {
            subtotal
        }
    }
}

// domain/Order.kt — aggregate root; no public constructor or data-class copy
class Order private constructor(
    val id: UUID,
    val customerId: String,
    private val lines: List<OrderItem>,
    val total: BigDecimal,
) {
    // A caller cannot mutate the aggregate through a returned collection.
    val items: List<OrderItem> get() = lines.toList()

    companion object {
        fun create(id: UUID, customerId: String, items: List<OrderItem>): Order {
            require(customerId.isNotBlank()) { "Customer ID is required" }
            require(items.isNotEmpty()) { "An order must contain an item" }
            val snapshot = items.toList()
            return Order(id, customerId, snapshot, OrderPricing.totalFor(snapshot))
        }
    }
}

// The adapter persists the complete aggregate as one atomic unit.
interface OrderRepository {
    suspend fun save(order: Order)
}

// application/CreateOrder.kt — orchestration, not domain object assembly
class CreateOrder(private val orders: OrderRepository) {
    suspend fun execute(customerId: String, items: List<OrderItem>): Order {
        val order = Order.create(UUID.randomUUID(), customerId, items)
        orders.save(order)
        return order
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
- [DDD Aggregate — Martin Fowler](https://martinfowler.com/bliki/DDD_Aggregate.html)
- [DDD Reference — Eric Evans](https://www.domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf) — Aggregates and Factories
