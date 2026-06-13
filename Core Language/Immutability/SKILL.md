---
name: immutability
description: >
  Immutability patterns and enforcement for Android/Kotlin development.
  Load this skill when designing data models, state classes, collections,
  or any shared data structure to ensure thread safety and predictable behavior.
---

# Immutability

## Overview

Immutability means that once an object is created, its state cannot change. In Android development, immutability is critical for thread safety, predictable UI state, and preventing hard-to-debug side effects.

---

## Core Principles

- Default to `val` — only use `var` when mutation is genuinely required
- Default to immutable collections — only use mutable when building
- State should flow in one direction — never mutate state from outside
- Shared data across threads must always be immutable

---

## Properties

```kotlin
// ✅ val by default
data class User(val id: String, val name: String, val email: String)

// ✅ var only when genuinely needed (e.g., local accumulator)
var count = 0
repeat(10) { count++ }

// ❌ var on a data class field shared across layers
data class User(var name: String)  // mutability leaks outside
```

---

## Data Classes and copy()

```kotlin
// ✅ Use copy() to produce new state — never mutate
data class UiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    val error: String? = null
)

// In ViewModel
_uiState.update { it.copy(isLoading = true) }
_uiState.update { it.copy(isLoading = false, items = newItems) }

// ❌ Never do this
_uiState.value.items = newItems  // not possible with val, but don't expose mutable state
```

---

## Collections

```kotlin
// ✅ Expose immutable collections from all public APIs
class UserRepository {
    fun getUsers(): List<User> = _users.toList()  // defensive copy
}

// ✅ Build with mutable, expose as immutable
val users: List<User> = buildList {
    add(User("1", "Ali"))
    add(User("2", "Sara"))
}

// ✅ StateFlow always holds immutable state
private val _uiState = MutableStateFlow(UiState())
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// ❌ Never expose MutableStateFlow or MutableList
val uiState = MutableStateFlow(UiState())  // external code can modify
```

---

## Immutable State in ViewModel

```kotlin
// ✅ Correct pattern — immutable state, controlled updates
class UserViewModel : ViewModel() {

    private val _state = MutableStateFlow(UserUiState())
    val state: StateFlow<UserUiState> = _state.asStateFlow()

    fun onNameChanged(name: String) {
        _state.update { it.copy(name = name) }
    }
}

// ✅ State class — all vals
data class UserUiState(
    val name: String = "",
    val isLoading: Boolean = false,
    val error: String? = null
)
```

---

## Immutability Across Layers

| Layer                   | Rule                                             |
| ----------------------- | ------------------------------------------------ |
| Domain models           | Always immutable (`data class` with `val`)       |
| DTOs                    | Always immutable (`data class` with `val`)       |
| UI State                | Always immutable — update via `copy()`           |
| Repository return types | Return `List<T>` not `MutableList<T>`            |
| Exposed flows           | Always `StateFlow`/`Flow` not `MutableStateFlow` |

---

## Thread Safety Through Immutability

```kotlin
// ✅ Immutable objects are safe to share across coroutines
data class Config(val baseUrl: String, val timeout: Long)

// Safe — no synchronization needed
val config = Config("https://api.example.com", 30_000)
launch(Dispatchers.IO) { fetchData(config) }
launch(Dispatchers.Default) { processData(config) }

// ❌ Mutable shared state requires synchronization
class Config {
    var baseUrl: String = ""  // race condition if accessed from multiple threads
}
```

---

## Anti-Patterns

- `var` properties on domain models or state classes
- Exposing `MutableList`, `MutableMap`, or `MutableStateFlow` from public APIs
- Mutating a list after exposing it — always return a defensive copy
- Using `apply {}` to mutate an object after it's been shared
- Casting `List<T>` to `MutableList<T>` — breaks immutability contract

---

## Related Skills

- `kotlin` — `val`/`var` and data class conventions
- `state-management` — UI state modeling with immutable data
- `coroutine` — thread safety with immutable shared state
- `repository-pattern` — immutable return types from repositories
