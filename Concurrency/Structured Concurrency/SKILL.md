---
name: structured-concurrency
description: >
  Structured concurrency principles and patterns in Kotlin coroutines.
  Load this skill when managing parent-child coroutine relationships,
  understanding cancellation propagation, using coroutineScope vs
  supervisorScope, or designing components with proper coroutine lifecycle.
---

# Structured Concurrency

## Overview

Structured concurrency ensures that coroutines have a defined lifetime tied to a scope. Every coroutine has a parent, and when the parent is cancelled, all children are cancelled. This prevents coroutine leaks and makes async code predictable and safe.

---

## Core Principles

- Every coroutine has a **parent scope** — no orphan coroutines
- Cancellation flows **downward** — parent cancelled → all children cancelled
- Failure flows **upward** — child exception cancels the parent (unless `SupervisorJob`)
- A scope does not complete until **all children complete**
- Use `coroutineScope` for all-or-nothing operations, `supervisorScope` for independent children

---

## Parent-Child Relationship

```
CoroutineScope (viewModelScope)
    └── Job (root)
         ├── Coroutine A (launch)
         │    └── Coroutine A1 (launch inside A)
         └── Coroutine B (launch)

Cancel viewModelScope → A, A1, B all cancelled
Exception in B → cancels scope → A, A1 also cancelled (unless SupervisorJob)
```

---

## coroutineScope vs supervisorScope

```kotlin
// ✅ coroutineScope — all children must succeed, any failure cancels all
suspend fun loadRequiredData(): DashboardData = coroutineScope {
    val user = async { userRepository.getUser() }    // required
    val config = async { configRepository.get() }    // required

    DashboardData(
        user = user.await(),      // if either throws, both cancelled
        config = config.await()
    )
}

// ✅ supervisorScope — children are independent, one failure doesn't cancel others
suspend fun loadOptionalData(): DashboardExtras = supervisorScope {
    val recommendations = async { recommendationRepository.get() }  // optional
    val ads = async { adsRepository.get() }                          // optional

    DashboardExtras(
        recommendations = runCatching { recommendations.await() }.getOrDefault(emptyList()),
        ads = runCatching { ads.await() }.getOrDefault(emptyList())
    )
}
```

---

## SupervisorJob in Custom Scopes

```kotlin
// ✅ SupervisorJob — child failures don't cancel the scope
@Singleton
class BackgroundSyncManager @Inject constructor() {

    // SupervisorJob: one sync failure doesn't cancel all other syncs
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun syncUsers() {
        scope.launch {
            userRepository.sync()  // failure here won't cancel syncOrders
        }
    }

    fun syncOrders() {
        scope.launch {
            orderRepository.sync()
        }
    }

    fun cancel() {
        scope.cancel()
    }
}

// ✅ viewModelScope already uses SupervisorJob internally
// Each launch{} in ViewModel is independent
```

---

## Cancellation Propagation

```kotlin
// ✅ CancellationException must NOT be caught and swallowed
suspend fun doWork() {
    try {
        delay(1_000)
        fetchData()
    } catch (e: CancellationException) {
        throw e  // ✅ always rethrow CancellationException
    } catch (e: Exception) {
        handleError(e)
    }
}

// ✅ runCatching re-throws CancellationException automatically
val result = runCatching { fetchData() }  // safe — cancellation propagates correctly

// ❌ Swallowing CancellationException — breaks structured concurrency
try {
    delay(1_000)
} catch (e: Exception) {
    // catches CancellationException too — coroutine won't cancel properly
}
```

---

## Scope Completion

```kotlin
// ✅ coroutineScope suspends until all children complete
suspend fun processAll(items: List<Item>) = coroutineScope {
    items.forEach { item ->
        launch { processItem(item) }
    }
    // resumes here only when all launches have completed
}

// ✅ joinAll — wait for specific jobs
suspend fun waitForAll() {
    val jobs = listOf(
        viewModelScope.launch { task1() },
        viewModelScope.launch { task2() }
    )
    jobs.joinAll()  // suspends until both complete
}
```

---

## Scope in Non-Lifecycle Components

```kotlin
// ✅ Inject and cancel scope explicitly in non-ViewModel classes
class DataSyncService @Inject constructor(
    private val repository: DataRepository
) {
    private val job = SupervisorJob()
    private val scope = CoroutineScope(job + Dispatchers.IO)

    fun start() {
        scope.launch {
            while (isActive) {
                repository.sync()
                delay(60_000)
            }
        }
    }

    fun stop() {
        job.cancel()  // cancels all children, scope is still reusable with new children
    }
}
```

---

## Anti-Patterns

- `GlobalScope.launch` — no parent, lives forever, can't be cancelled
- Catching `CancellationException` without rethrowing — breaks cancellation
- Creating a `CoroutineScope` without `SupervisorJob` in a manager class — one failure kills all tasks
- Not calling `scope.cancel()` in `onCleared()` or equivalent — coroutine leak
- Using `launch` inside `suspend fun` without a scope — use `coroutineScope {}` instead

---

## Related Skills

- `coroutine` — coroutine fundamentals and dispatcher selection
- `flow` — structured cancellation with flows
- `lifecycle` — viewModelScope and lifecycleScope internals
- `background-processing` — long-running work with structured scopes
