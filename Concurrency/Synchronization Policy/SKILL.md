---
name: synchronization-policy
description: >
  Thread safety and synchronization policy for Android with coroutines.
  Load this skill when defining how shared state is accessed across threads,
  deciding between confinement vs locking, ensuring data consistency,
  or auditing code for race conditions.
---

# Synchronization Policy

## Overview

Synchronization policy defines the rules for safely accessing shared mutable state across coroutines and threads. The safest approach is **confinement** — restricting access to a single thread or coroutine — rather than locking. When shared access is unavoidable, use coroutine-friendly primitives like `Mutex` and `StateFlow.update`.

---

## Core Principles

- Prefer **confinement** over locking — restrict state to one dispatcher or coroutine
- Prefer **immutable data** — no synchronization needed if data can't change
- Use `StateFlow.update` for UI state — it's atomic and coroutine-safe
- Use `Mutex` only when state must be shared across multiple coroutines
- Never access the same mutable state from both `Dispatchers.IO` and `Dispatchers.Main` without synchronization

---

## Strategy 1 — Confinement (Preferred)

```kotlin
// ✅ Confine mutable state to a single dispatcher
class DataCache @Inject constructor() {
    // All access happens on a single-threaded dispatcher
    private val cacheDispatcher = Dispatchers.IO.limitedParallelism(1)
    private val cache = mutableMapOf<String, Data>()

    suspend fun get(key: String): Data? = withContext(cacheDispatcher) {
        cache[key]
    }

    suspend fun put(key: String, data: Data) = withContext(cacheDispatcher) {
        cache[key] = data
    }

    suspend fun clear() = withContext(cacheDispatcher) {
        cache.clear()
    }
}
```

---

## Strategy 2 — Immutable State + Atomic Updates

```kotlin
// ✅ Immutable data class + atomic StateFlow update
class CartManager @Inject constructor() {
    private val _cart = MutableStateFlow(Cart())
    val cart: StateFlow<Cart> = _cart.asStateFlow()

    fun addItem(item: CartItem) {
        _cart.update { current ->
            current.copy(items = current.items + item)  // new list — immutable
        }
    }

    fun removeItem(itemId: String) {
        _cart.update { current ->
            current.copy(items = current.items.filter { it.id != itemId })
        }
    }
}

data class Cart(
    val items: List<CartItem> = emptyList()  // immutable list
)
```

---

## Strategy 3 — Mutex for Shared Mutable State

```kotlin
// ✅ Use Mutex when multiple coroutines must share mutable state
class ConnectionPool @Inject constructor() {
    private val mutex = Mutex()
    private val connections = mutableListOf<Connection>()

    suspend fun acquire(): Connection = mutex.withLock {
        connections.removeFirstOrNull() ?: createNewConnection()
    }

    suspend fun release(connection: Connection) = mutex.withLock {
        connections.add(connection)
    }
}
```

---

## Choosing the Right Strategy

```
Is the state only read after initialization?
  → Use val + immutable data class — no sync needed

Is the state only accessed from one coroutine/dispatcher?
  → Use confinement (limitedParallelism(1)) — no sync needed

Is the state a simple counter or flag?
  → Use AtomicInteger / AtomicBoolean — no coroutine sync needed

Is the state UI state in a ViewModel?
  → Use MutableStateFlow.update — built-in atomic update

Is the state accessed by multiple coroutines concurrently?
  → Use Mutex.withLock
```

---

## Thread Confinement with limitedParallelism

```kotlin
// ✅ Single-threaded context for all operations on shared resource
private val singleThreadContext = Dispatchers.IO.limitedParallelism(1)

// All database writes serialized through single thread
suspend fun writeData(data: List<Item>) = withContext(singleThreadContext) {
    dao.deleteAll()
    dao.insertAll(data)
}
```

---

## Common Race Conditions to Avoid

```kotlin
// ❌ Check-then-act race condition
if (cache[key] == null) {           // thread A checks
    cache[key] = expensiveCompute() // thread B also checks — both compute
}

// ✅ Fix with Mutex
mutex.withLock {
    if (cache[key] == null) {
        cache[key] = expensiveCompute()
    }
}

// ❌ Non-atomic read-modify-write
_state.value = _state.value.copy(count = _state.value.count + 1)  // race

// ✅ Fix with update
_state.update { it.copy(count = it.count + 1) }  // atomic
```

---

## Annotations for Documentation

```kotlin
// ✅ Document thread-safety expectations
@ThreadSafe
class SafeCache { /* ... */ }

@MainThread
fun updateUi(data: List<Item>) { /* ... */ }

@WorkerThread
suspend fun loadFromDisk(): List<Item> { /* ... */ }
```

---

## Anti-Patterns

- Accessing a `MutableList` from multiple coroutines without synchronization — race condition
- Using `synchronized {}` in coroutine code — blocks the thread
- Mutable `var` property in a class accessed from multiple dispatchers without protection
- Reading `StateFlow.value` and writing separately — use `update {}` instead
- Assuming coroutines on the same dispatcher are sequential — they may interleave at suspension points

---

## Related Skills

- `mutex-strategy` — Mutex and Semaphore implementation details
- `coroutine` — dispatcher selection and coroutine fundamentals
- `structured-concurrency` — coroutine lifetime and cancellation
- `stateflow` — atomic state updates with StateFlow
