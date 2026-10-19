---
title: Enforce Separation of Concerns Between Layers
paths:
  - "src/**/*"
impact: HIGH
---

# Enforce Separation of Concerns Between Layers

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Do NOT mix business logic with UI components or place database queries directly in controllers. Each architectural layer must have a single responsibility: controllers handle HTTP concerns, services encapsulate business logic, and repositories manage data access. Violating these boundaries creates tightly coupled code that is difficult to test, refactor, and reason about. When business rules live inside controllers, they cannot be reused across different entry points (API, CLI, events) and changes to infrastructure leak into domain logic. Maintain clear boundaries between contexts by delegating work through well-defined interfaces rather than inlining cross-cutting concerns.

## Critical principles

- Do NOT mix business logic with UI components
- Keep database queries out of controllers
- Maintain clear boundaries between contexts
- Ensure proper separation of responsibilities

## Incorrect

The controller mixes HTTP handling, business logic, and database access in a single function, making it impossible to reuse or test the business rules independently. `OrderDatabase` represents the application's database adapter; request and response types carry HTTP data.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

// OrderController.kt — everything in one place
class OrderController(private val database: OrderDatabase) {
    suspend fun createOrder(request: CreateOrderRequest): OrderResponse {
        // Database access directly in the controller
        val customer = database.findCustomer(request.customerId)
            ?: return OrderResponse(status = 404, error = "Customer not found")

        // Business logic mixed into the controller
        var total = BigDecimal.ZERO
        for (item in request.items) {
            val product = database.findProduct(item.productId)
                ?: throw NotFoundException("Product not found")
            total += product.price * item.quantity.toBigDecimal()
        }
        if (total > BigDecimal("10000.00")) {
            total = (total * BigDecimal("0.90")).setScale(2, RoundingMode.HALF_UP)
        }

        // More database access inline
        val order = database.insertOrder(CreateOrderData(customer.id, total))
        return OrderResponse(status = 201, body = order)
    }
}
```

## Correct

The controller delegates to a service for business logic and a repository for data access. Each layer has a single responsibility and can be tested and reused independently.

```kotlin
import java.math.BigDecimal
import java.math.RoundingMode

// OrderController.kt — handles HTTP only
class OrderController(private val orderService: OrderService) {
    suspend fun createOrder(request: CreateOrderRequest): OrderResponse = try {
        val order = orderService.createOrder(request.customerId, request.items)
        OrderResponse(status = 201, body = order)
    } catch (error: NotFoundException) {
        OrderResponse(status = 404, error = error.message)
    }
}

// OrderService.kt — business logic only
class OrderService(
    private val customers: CustomerRepository,
    private val products: ProductRepository,
    private val orders: OrderRepository,
) {
    suspend fun createOrder(customerId: String, items: List<OrderItem>): Order {
        val customer = customers.findById(customerId)
            ?: throw NotFoundException("Customer not found")
        val total = calculateTotal(items)
        return orders.create(CreateOrderData(customer.id, total))
    }

    private suspend fun calculateTotal(items: List<OrderItem>): BigDecimal {
        var total = BigDecimal.ZERO
        for (item in items) {
            val product = products.findById(item.productId)
                ?: throw NotFoundException("Product not found")
            total += product.price * item.quantity.toBigDecimal()
        }
        return if (total > BigDecimal("10000.00")) {
            (total * BigDecimal("0.90")).setScale(2, RoundingMode.HALF_UP)
        } else {
            total
        }
    }
}

// OrderRepository.kt — data-access contract
interface OrderRepository {
    suspend fun create(data: CreateOrderData): Order
}

// SqlOrderRepository.kt — data access only
class SqlOrderRepository(private val database: OrderDatabase) : OrderRepository {
    override suspend fun create(data: CreateOrderData): Order =
        database.insertOrder(data)
}
```

## Reference

- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
