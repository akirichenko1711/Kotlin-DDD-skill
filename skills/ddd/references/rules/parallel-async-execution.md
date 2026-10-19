---
title: Parallelize Independent Async Operations
paths:
  - "src/**/*"
---

# Parallelize Independent Async Operations

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Independent asynchronous operations that share no data dependency must run concurrently, not sequentially. Sequential execution adds each operation's latency together. Concurrent execution reduces total latency to the slowest single operation.

This applies to I/O-bound work such as HTTP requests, database queries, and cache lookups. In Kotlin, use structured concurrency with `coroutineScope` and `async` for independent suspending calls. Declaring a function `suspend` does not make blocking I/O non-blocking; retain the project's dispatcher and client conventions.

The examples assume application ports `DataService` and `ItemApi` with suspending `getData(key): Data` and `fetchItem(id): Item` methods. `Data` and `Item` are application value types.

## How to identify violations

Scan for these patterns:

- Consecutive suspending calls where the second does **not** read from the first.
- A loop that completes one independent suspending operation per iteration instead of starting independent work together.
- Sequential calls to different services, different endpoints, or different keys against the same service.
- "Warm-up" or "prefetch" steps performed serially before the main work begins.

## Incorrect

Two independent service calls awaited one after the other. Total latency = `latency(A) + latency(B)`.

```kotlin
suspend fun loadData(
    key: String,
    serviceA: DataService,
    serviceB: DataService,
): Pair<Data, Data> {
    val dataA = serviceA.getData(key)
    val dataB = serviceB.getData(key)
    return dataA to dataB
}
```

A loop awaiting one independent request at a time. Total latency = `N * latency(request)`.

```kotlin
suspend fun loadItems(ids: List<String>, api: ItemApi): List<Item> {
    val results = mutableListOf<Item>()
    for (id in ids) {
        results += api.fetchItem(id)
    }
    return results
}
```

## Correct

Both calls are started before either result is awaited. With non-blocking clients and sufficient resources, total latency approaches `max(latency(A), latency(B))`.

```kotlin
import kotlinx.coroutines.async
import kotlinx.coroutines.coroutineScope

suspend fun loadData(
    key: String,
    serviceA: DataService,
    serviceB: DataService,
): Pair<Data, Data> = coroutineScope {
    val dataA = async { serviceA.getData(key) }
    val dataB = async { serviceB.getData(key) }
    dataA.await() to dataB.await()
}
```

For a small bounded batch and a concurrent-safe client, start all requests and collect their results in input order. Large batches require bounded concurrency rather than one coroutine per item.

```kotlin
import kotlinx.coroutines.async
import kotlinx.coroutines.awaitAll
import kotlinx.coroutines.coroutineScope

suspend fun loadItems(ids: List<String>, api: ItemApi): List<Item> =
    coroutineScope {
        ids.map { id -> async { api.fetchItem(id) } }.awaitAll()
    }
```


## When NOT to apply

- **Data dependency.** A later operation reads from an earlier operation's result.
- **Ordering or causality.** The operations must observably happen in a specific order (e.g., write-then-read against a non-linearizable store).
- **Rate limits, quotas, or backpressure.** Parallelism would exceed downstream limits; prefer a bounded worker pool or semaphore instead of unbounded concurrency.
- **Resource contention.** The operations contend for a single connection, lock, or transaction.
- **Failure-handling differences.** `coroutineScope` cancels sibling children when a child fails. For partial per-item outcomes, design explicit failure handling with `supervisorScope`; it does not turn `awaitAll` into an all-settled operation. Always propagate `CancellationException`.

## Reference

- [Composing suspending functions](https://kotlinlang.org/docs/composing-suspending-functions.html) — Kotlin documentation
