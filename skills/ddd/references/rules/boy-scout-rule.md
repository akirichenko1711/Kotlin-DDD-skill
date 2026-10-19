---
title: Boy Scout Rule — Incremental Improvement Without Scope Creep
paths:
  - "src/**/*"
---

# Boy Scout Rule — Incremental Improvement Without Scope Creep

> Modified 2026-10-19: examples and language-specific notes adapted to Kotlin from NeoLabHQ's DDD rules (GPL-3.0-only).

Rules: 
- Leave code better than you found it (Martin, "Clean Code") — but limit improvements to the code you are already touching. 
- Apply Opportunistic Refactoring (Fowler): make small cleanups while working on a task, not as a separate effort. Stop when the improvement is unrelated to your current change. 
- Over-engineering disguised as "cleaning up" violates YAGNI (Beck & Jeffries, "Extreme Programming") and expands scope, making changes harder to review, test, and revert.

**Appropriate improvements** when touching a function: rename unclear variables, add missing type annotations, extract a small helper, remove dead code, fix an obvious code smell.

**Not appropriate** when fixing a bug or adding a feature: restructuring entire modules, introducing new design patterns, refactoring code in files you are not otherwise modifying, replacing working implementations with "better" alternatives.


## Example

Task is to fix a null-check bug in `getUser`. Agent also restructures the module, renames unrelated functions, and introduces a new pattern — turning a one-line fix into a large, unrelated refactor.

### Before

Starting code: `getUser` can crash when `user.profile` is missing

```kotlin
// Task: fix bug — getUser crashes when user has no profile
suspend fun getUser(id: String): User {
    val user = userRepo.findById(id) ?: throw NotFoundException("User")
    // Bug: assumes profile always exists
    val r = user.copy(displayName = formatName(user.name, user.profile!!))
    return r
}
```

### Incorrect

Agent rewrites the whole module instead of fixing the bug in place.

```kotlin
// Task: fix bug — getUser crashes when user has no profile

// Agent introduces a new result hierarchy for a local null-check fix.
sealed interface UserLookupResult {
    data class Found(val user: User) : UserLookupResult
    data object Missing : UserLookupResult
}

// Renamed from getUser to fetchUserWithProfile (unrelated change).
suspend fun fetchUserWithProfile(id: String): UserDto {
    val result = userRepo.findById(id)?.let { UserLookupResult.Found(it) }
        ?: UserLookupResult.Missing
    val user = when (result) {
        is UserLookupResult.Found -> enrichWithProfile(result.user)
        UserLookupResult.Missing -> throw NotFoundException("User")
    }
    // Introduced a DTO mapper (unrelated change).
    return UserMapper.toDto(user)
}

private fun enrichWithProfile(user: User): User {
    val profile = user.profile ?: DEFAULT_PROFILE
    return user.copy(profile = profile, displayName = formatName(user.name, profile))
}

// Refactored other functions not related to the bug.
suspend fun listUsers(): List<UserDto> =
    userRepo.findAll().map { UserMapper.toDto(enrichWithProfile(it)) }

suspend fun deleteUser(id: String) = userRepo.removeById(id)
```

### Correct

Agent fixes the bug and makes only small, adjacent improvements to the code it already touches.

```kotlin
// Task: fix bug — getUser crashes when user has no profile
suspend fun getUser(id: String): User {
    val user = userRepo.findById(id) ?: throw NotFoundException("User")

    // Bug fix: guard against missing profile.
    val profile = user.profile ?: DEFAULT_PROFILE

    // Boy scout: remove an unclear temporary variable in the code being changed.
    return user.copy(profile = profile, displayName = formatName(user.name, profile))
}
```

## Reference

- [Clean Code — Robert C. Martin](https://www.oreilly.com/library/view/clean-code/9780136083238/) — Boy Scout Rule: "Leave the campground cleaner than you found it"
- [Opportunistic Refactoring — Martin Fowler](https://martinfowler.com/bliki/OpportunisticRefactoring.html) — "Refactor as you go, not as a separate activity"
- [Extreme Programming Explained — Kent Beck & Ron Jeffries](https://www.oreilly.com/library/view/extreme-programming-explained/0201616416/) — YAGNI: "You Aren't Gonna Need It"
