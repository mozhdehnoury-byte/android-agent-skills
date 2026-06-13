---
name: kotlin
description: >
  Kotlin language conventions and best practices for Android development.
  Load this skill whenever writing, reviewing, or refactoring any Kotlin code.
  Covers idioms, language features, and patterns that should be consistently
  applied across the entire codebase.
---

# Kotlin

## Overview

Kotlin is the primary language for Android development. This skill defines which language features to use, when to use them, and which patterns to avoid.

---

## Core Principles

- Prefer **idiomatic Kotlin** over Java-style code
- Prefer **immutability** by default (`val` over `var`, `data class`, `copy()`)
- Prefer **null safety** at compile time — minimize nullable types
- Prefer **expression over statement** where it improves readability
- Never use `!!` — always handle nullability explicitly

---

## Language Features

### Null Safety

```kotlin
// ✅ Use safe call + elvis
val name = user?.name ?: "Unknown"

// ✅ Use let for nullable execution
user?.let { sendEmail(it) }

// ❌ Never force-unwrap
val name = user!!.name
```

### Data Classes

```kotlin
// ✅ Use data class for value holders
data class User(val id: String, val name: String)

// ✅ Use copy() for mutation
val updated = user.copy(name = "Ali")

// ❌ Never mutate fields directly
user.name = "Ali"
```

### Sealed Classes

```kotlin
// ✅ Use sealed class for exhaustive state modeling
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: Throwable) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

// ✅ Always use when exhaustively — no else branch
when (result) {
    is Result.Success -> showData(result.data)
    is Result.Error   -> showError(result.exception)
    is Result.Loading -> showLoading()
}
```

### Extension Functions

```kotlin
// ✅ Use for adding behavior to existing types without inheritance
fun String.toFormattedDate(): String { ... }

// ✅ Use for scoping utility functions to a receiver
fun Context.showToast(message: String) {
    Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
}

// ❌ Don't use extension functions to bypass encapsulation
// ❌ Don't define extensions on types you own — add the method directly
```

### Scope Functions

```kotlin
// ✅ let  — nullable transformation or scoped execution
val length = text?.let { it.trim().length }

// ✅ apply — object configuration / builder pattern
val intent = Intent().apply {
    putExtra("key", value)
    flags = Intent.FLAG_ACTIVITY_NEW_TASK
}

// ✅ run   — transformation on receiver
val result = user.run { "$name ($email)" }

// ✅ also  — side effects (logging, debugging)
val user = createUser().also { log("User created: $it") }

// ❌ Don't nest scope functions more than 2 levels deep
// ❌ Don't use scope functions just for style — only when they add clarity
```

### When Expression

```kotlin
// ✅ Use when as an expression
val label = when (status) {
    Status.ACTIVE   -> "Active"
    Status.INACTIVE -> "Inactive"
    Status.PENDING  -> "Pending"
}

// ✅ Always exhaustive — avoid else on sealed/enum
// ❌ Don't use else on sealed class when — it hides new state cases
```

### Inline Classes / Value Classes

```kotlin
// ✅ Use to add type safety to primitives
@JvmInline
value class UserId(val value: String)

@JvmInline
value class Email(val value: String)

// Prevents mixing up primitive parameters
fun findUser(id: UserId, email: Email) { ... }
```

### Collections

```kotlin
// ✅ Use immutable collections by default
val items: List<String> = listOf("a", "b", "c")
val map: Map<String, Int> = mapOf("a" to 1)

// ✅ Use mutableListOf() only when mutation is required
val buffer = mutableListOf<String>()

// ✅ Prefer functional operators
val names = users.filter { it.isActive }.map { it.name }

// ❌ Don't use Java stream API — use Kotlin collections API
```

---

## Naming Conventions

| Element  | Convention                 | Example                    |
| -------- | -------------------------- | -------------------------- |
| Class    | PascalCase                 | `UserRepository`           |
| Function | camelCase                  | `getUserById()`            |
| Property | camelCase                  | `userName`                 |
| Constant | SCREAMING_SNAKE            | `MAX_RETRY_COUNT`          |
| Package  | lowercase                  | `com.example.feature.auth` |
| File     | PascalCase (matches class) | `UserRepository.kt`        |

---

## Anti-Patterns

- `!!` — use safe calls and elvis instead
- `lateinit var` on types that could be `val` — redesign initialization
- `object` for classes that hold state — use class with DI instead
- Returning `null` to signal error — use `Result<T>` or sealed class
- Java-style getters/setters — use Kotlin properties
- `companion object` holding mutable state — use DI or top-level properties
- Catching `Exception` broadly — catch specific exception types

---

## Related Skills

- `coroutine` — async/concurrency patterns
- `extension-functions-design` — detailed extension function guidelines
- `immutability` — immutability policy across layers
- `serialization` — Kotlin serialization setup and usage
