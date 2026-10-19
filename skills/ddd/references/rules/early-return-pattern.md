---
title: Use Early Returns to Reduce Nesting
impact: MEDIUM
paths:
  - "**/*"
  - "**/*"
---

# Use Early Returns to Reduce Nesting

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Always use early returns to handle error conditions and edge cases at the top of functions instead of wrapping logic in nested conditionals. Deeply nested code (more than 3 levels) increases cognitive load, obscures the happy path, and makes functions harder to read, review, and maintain. When guard clauses are placed first, the main logic stays at the top indentation level and reads linearly from top to bottom.

## Incorrect

Validation checks are nested inside each other, pushing the core business logic deep into indentation. The happy path is buried at the innermost level, and error handling is scattered across multiple `else` branches at the bottom.

```kotlin
suspend fun validateUser(userId: String, role: String): User {
    if (userId.isNotEmpty()) {
        val user = db.users.findById(userId)
        if (user != null) {
            if (!user.isDeleted) {
                if (user.role == role) {
                    if (user.emailVerified) {
                        // happy path buried 5 levels deep
                        return user
                    } else {
                        throw IllegalStateException("Email not verified")
                    }
                } else {
                    throw IllegalStateException("Insufficient role")
                }
            } else {
                throw IllegalStateException("User is deleted")
            }
        } else {
            throw IllegalStateException("User not found")
        }
    } else {
        throw IllegalArgumentException("User ID is required")
    }
}
```

## Correct

Guard clauses handle each error condition with an early exit at the top level. The happy path flows naturally at the end of the function with zero unnecessary nesting.

```kotlin
suspend fun validateUser(userId: String, role: String): User {
    if (userId.isEmpty()) {
        throw IllegalArgumentException("User ID is required")
    }

    val user = db.users.findById(userId)
        ?: throw IllegalStateException("User not found")
    if (user.isDeleted) {
        throw IllegalStateException("User is deleted")
    }
    if (user.role != role) {
        throw IllegalStateException("Insufficient role")
    }
    if (!user.emailVerified) {
        throw IllegalStateException("Email not verified")
    }

    return user
}
```

## Reference

- [Flattening Arrow Code](https://blog.codinghorror.com/flattening-arrow-code/)
