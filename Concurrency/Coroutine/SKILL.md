---
name: coroutine
description: >
  Kotlin Coroutines fundamentals and patterns for Android development.
  Load this skill when launching coroutines, choosing the right scope,
  managing coroutine lifecycle, handling exceptions, or structuring
  async operations with suspend functions.
---

# Coroutine

## Overview

Kotlin Coroutines provide a structured way to write asynchronous code sequentially. On Android, coroutines are the standard for all async operations — network calls, database queries, and background work. Every coroutine must be launched in a scope that controls its lifecycle.

---

## Core Principles

- Always use the **narrowest scope** available — `viewModelScope`, `lifecycleScope`, not `GlobalScope`
- Use `Dispatchers.IO` for blocking I/O, `Dispatchers.Default` for CPU-heavy work
- Never use `GlobalScope` — it has no lifecycle and can't be cancelled
- Exceptions in coroutines must be handled explicitly — they don't propagate like regular exceptions
- Use `supervisorScope` when child failures should not cancel siblings

---

## Scope Selection

```kotlin
// ✅ viewModelScope — cancelled when ViewModel is cleared
class UserViewModel : ViewModel() {
    fun loadUser() {
        viewModelScope.launch {
            // auto-cancelled when ViewModel is destroyed
        }
    }
}

// ✅ lifecycleScope — cancelled when lifecycle is destroyed
class UserFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // cancelled when below STARTED
            }
        }
    }
}

// ✅ Custom scope — for non-lifecycle components
@Singleton
class SyncManager @Inject constructor() {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun startSync() {
        scope.launch { /* ... */ }
    }

    fun cancel() {
        scope.cancel()
    }
}

// ❌ Never use GlobalScope
GlobalScope.launch { /* no lifecycle, no cancellation */ }
```

---

## Dispatcher Selection

```kotlin
// ✅ IO — network, database, file operations
viewModelScope.launch(Dispatchers.IO) {
    val users = userDao.getAll()
}

// ✅ Default — CPU-intensive work (sorting, parsing, computation)
viewModelScope.launch(Dispatchers.Default) {
    val sorted = largeList.sortedBy { it.name }
}

// ✅ Main — UI updates (usually implicit in viewModelScope)
viewModelScope.launch(Dispatchers.Main) {
    binding.textView.text = "Updated"
}

// ✅ withContext — switch dispatcher inside a coroutine
suspend fun processData(): List<Item> = withContext(Dispatchers.Default) {
    rawData.map { parse(it) }
}
```

---

## Exception Handling

```kotlin
// ✅ try/catch inside launch
viewModelScope.launch {
    try {
        val user = repository.getUser(id)
        _state.value = UiState.Success(user)
    } catch (e: Exception) {
        _state.value = UiState.Error(e.message ?: "Error")
    }
}

// ✅ CoroutineExceptionHandler for unhandled exceptions
val handler = CoroutineExceptionHandler { _, throwable ->
    Timber.e(throwable, "Unhandled coroutine exception")
}

viewModelScope.launch(handler) {
    riskyOperation()
}

// ✅ runCatching — functional style
val result = runCatching { repository.getUser(id) }
result.onSuccess { _state.value = UiState.Success(it) }
result.onFailure { _state.value = UiState.Error(it.message ?: "Error") }

// ❌ async exception — not caught by try/catch on launch
val deferred = viewModelScope.async { riskyOperation() }
deferred.await()  // exception thrown here — wrap in try/catch
```

---

## Parallel Execution

```kotlin
// ✅ Run multiple operations in parallel with async
suspend fun loadDashboard(): Dashboard {
    return coroutineScope {
        val users = async { userRepository.getUsers() }
        val orders = async { orderRepository.getOrders() }
        val stats = async { statsRepository.getStats() }

        Dashboard(
            users = users.await().getOrDefault(emptyList()),
            orders = orders.await().getOrDefault(emptyList()),
            stats = stats.await().getOrNull()
        )
    }
}

// ✅ supervisorScope — one failure doesn't cancel others
suspend fun loadOptionalData() = supervisorScope {
    val primary = async { primaryRepository.getData() }
    val secondary = async { secondaryRepository.getData() }  // failure here won't cancel primary

    PrimaryData(
        main = primary.await(),
        extra = runCatching { secondary.await() }.getOrNull()
    )
}
```

---

## Cancellation

```kotlin
// ✅ Check cancellation in long loops
suspend fun processItems(items: List<Item>) {
    items.forEach { item ->
        ensureActive()  // throws CancellationException if cancelled
        processItem(item)
    }
}

// ✅ withTimeout
suspend fun fetchWithTimeout(): User = withTimeout(5_000) {
    api.getUser(id)
}

// ✅ withTimeoutOrNull — returns null on timeout
val user = withTimeoutOrNull(5_000) { api.getUser(id) }
```

---

## Anti-Patterns

- Using `GlobalScope` — no lifecycle management, can't be cancelled
- Launching coroutines without handling exceptions — silent failures
- Calling blocking functions (`Thread.sleep`, blocking I/O) without `Dispatchers.IO`
- Using `runBlocking` in Android code outside of tests — blocks the thread
- Ignoring `CancellationException` in catch blocks — breaks structured concurrency

---

## Related Skills

- `flow` — reactive streams with coroutines
- `structured-concurrency` — parent-child coroutine relationships
- `viewmodel` — viewModelScope usage
- `lifecycle` — lifecycleScope and repeatOnLifecycle
