---
title: Function and File Size Limits
impact: MEDIUM
paths:
  - "**/*"
  - "**/*"
---

# Function and File Size Limits

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

- Decompose functions longer than 80 lines into smaller, focused functions of 50 lines or fewer. When a function grows beyond 80 lines, it is almost certainly doing more than one thing and should be split. 
- Keep files under 200 lines of code. Large functions accumulate multiple responsibilities, making them harder to test, review, and reuse. 
- Extract cohesive blocks of logic into named functions that each serve a single purpose. If extracted functions are only used within the same context, keep them in the same file. However, when a file exceeds 200 lines even after decomposition, split related functions into separate modules grouped by responsibility.

## Incorrect

A single function handles validation, transformation, persistence, and notification. The condensed example below shows the responsibilities that can make a real implementation grow beyond 80 lines and become difficult to test or reuse independently. The repository, password hasher, and notification services are application-owned ports; `clock` is a `java.time.Clock`.

```kotlin
suspend fun processUserRegistration(input: Any?): User {
    // Validate input.
    require(input is Map<*, *>) { "Invalid input" }
    val email = input["email"] as? String ?: throw IllegalArgumentException("Email required")
    val name = input["name"] as? String ?: throw IllegalArgumentException("Name required")
    val password = input["password"] as? String ?: throw IllegalArgumentException("Password required")
    val role = input["role"] as? String
    require(email.isNotEmpty()) { "Email required" }
    require(name.isNotEmpty()) { "Name required" }
    require(password.isNotEmpty()) { "Password required" }
    require(password.length >= 8) { "Password too short" }
    require(password.any { it in 'A'..'Z' }) { "Password needs uppercase" }
    require(password.any { it in '0'..'9' }) { "Password needs digit" }
    require(Regex("""^[^\s@]+@[^\s@]+\.[^\s@]+$""").matches(email)) { "Invalid email format" }

    // Normalize data.
    val normalizedEmail = email.lowercase().trim()
    val normalizedName = name.trim().replace(Regex("""\s+"""), " ")
    val hashedPassword = passwordHasher.hash(password, cost = 12)
    val assignedRole = role?.takeUnless { it == "admin" || it.isEmpty() } ?: "user"
    val createdAt = clock.instant()
    val updatedAt = createdAt

    // Check duplicates and persist.
    val existing = users.findByEmail(normalizedEmail)
    check(existing == null) { "Email already registered" }
    val user = users.create(
        NewUser(
            email = normalizedEmail,
            name = normalizedName,
            passwordHash = hashedPassword,
            role = assignedRole,
            createdAt = createdAt,
            updatedAt = updatedAt,
        ),
    )

    // Send notifications.
    emailService.sendWelcome(normalizedEmail, normalizedName)
    analyticsService.trackRegistration(user.id, assignedRole, createdAt)
    auditLog.recordRegistration(user.id, normalizedEmail)

    return user
}
```

## Correct

Each responsibility is extracted into a focused function under 50 lines. Functions that are only used together stay in the same file.

```kotlin
fun validateRegistrationInput(input: Any?): RegistrationInput {
    require(input is Map<*, *>) { "Invalid input" }
    val email = input["email"] as? String ?: throw IllegalArgumentException("Email required")
    val name = input["name"] as? String ?: throw IllegalArgumentException("Name required")
    val password = input["password"] as? String ?: throw IllegalArgumentException("Password required")
    val role = input["role"] as? String
    require(email.isNotEmpty()) { "Email required" }
    require(name.isNotEmpty()) { "Name required" }
    require(password.isNotEmpty()) { "Password required" }
    require(password.length >= 8) { "Password too short" }
    require(password.any { it in 'A'..'Z' }) { "Password needs uppercase" }
    require(password.any { it in '0'..'9' }) { "Password needs digit" }
    require(Regex("""^[^\s@]+@[^\s@]+\.[^\s@]+$""").matches(email)) { "Invalid email format" }
    return RegistrationInput(email, name, password, role)
}

suspend fun normalizeAndHash(input: RegistrationInput): NormalizedUser = NormalizedUser(
    email = input.email.lowercase().trim(),
    name = input.name.trim().replace(Regex("""\s+"""), " "),
    passwordHash = passwordHasher.hash(input.password, cost = 12),
    role = input.role?.takeUnless { it == "admin" || it.isEmpty() } ?: "user",
)

suspend fun persistUser(data: NormalizedUser): User {
    check(users.findByEmail(data.email) == null) { "Email already registered" }
    val now = clock.instant()
    return users.create(
        NewUser(data.email, data.name, data.passwordHash, data.role, now, now),
    )
}

suspend fun notifyRegistration(user: User) {
    emailService.sendWelcome(user.email, user.name)
    analyticsService.trackRegistration(user.id, user.role, user.createdAt)
    auditLog.recordRegistration(user.id, user.email)
}

suspend fun processUserRegistration(input: Any?): User {
    val validated = validateRegistrationInput(input)
    val normalized = normalizeAndHash(validated)
    val user = persistUser(normalized)
    notifyRegistration(user)
    return user
}
```
