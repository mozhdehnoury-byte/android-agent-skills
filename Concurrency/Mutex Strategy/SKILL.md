---
name: mutex-strategy
description: >
  Mutex and coroutine-based locking strategies in Android.
  Load this skill when protecting shared mutable state across coroutines,
  preventing concurrent access to a critical section, serializing operations,
  or avoiding race conditions in async code.
---

# Mutex Strategy

## Overview
`Mutex` is a coroutine-friendly mutual exclusion lock. Unlike Java's `synchronized`, it suspends rather than blocks the thread while waiting for the lock. It is the correct tool for protecting shared mutable state that multiple coroutines may access concurrently.

---

## Core Principles

- Use `Mutex` instead of `synchronized` in coroutine code — `synchronized` blocks the thread
- Keep the critical section **as short as possible** — don't do heavy work inside `withLock`
- Never `withLock` on the main dispatcher for blocking operations
- Use `Mutex` for single-resource protection — use `Semaphore` for limiting concurrency count
- Prefer immutable state + atomic updates where possible — avoid Mutex if you can

---

## Basic Mutex Usage

```kotlin
// ✅ Protect shared mutable state
class TokenCache @Inject constructor() {
    private val mutex = Mutex()
    private var cachedToken: String? = null
    private var tokenExpiry: Long = 0L

    suspend fun getToken(fetchToken: suspend () -> String): String {
        mutex.withLock {
            val now = System.currentTimeMillis()
            if (cachedToken != null && now < tokenExpiry) {
                return cachedToken!!
            }
            val newToken = fetchToken()
            cachedToken = newToken
            tokenExpiry = now + 3_600_000L  // 1 hour
            return newToken
        }
    }

    suspend fun invalidate() {
        mutex.withLock {
            cachedToken = null
            tokenExpiry = 0L
        }
    }
}
```

---

## Token Refresh with Mutex

```kotlin
// ✅ Prevent multiple simultaneous token refreshes
class AuthRepository @Inject constructor(
    private val api: AuthApi,
    private val tokenStorage: TokenStorage
) {
    private val refreshMutex = Mutex()

    suspend fun refreshToken(): String? {
        return refreshMutex.withLock {
            // Check if another coroutine already refreshed while we were waiting
            val currentToken = tokenStorage.getAccessToken()
            if (currentToken != null && !isTokenExpired(currentToken)) {
                return@withLock currentToken
            }

            runCatching {
                val response = api.refresh(tokenStorage.getRefreshToken() ?: return@withLock null)
                tokenStorage.saveTokens(response.accessToken, response.refreshToken)
                response.accessToken
            }.getOrNull()
        }
    }
}
```

---

## Semaphore — Limit Concurrency

```kotlin
// ✅ Semaphore — allow max N concurrent operations
class ImageProcessor @Inject constructor() {
    // Max 3 concurrent image processing operations
    private val semaphore = Semaphore(3)

    suspend fun processImage(imageUri: Uri): Bitmap {
        return semaphore.withPermit {
            heavyImageProcessing(imageUri)
        }
    }
}

// ✅ Rate-limited API calls
class ApiThrottler(maxConcurrent: Int = 5) {
    private val semaphore = Semaphore(maxConcurrent)

    suspend fun <T> throttled(call: suspend () -> T): T {
        return semaphore.withPermit { call() }
    }
}
```

---

## Mutex vs Other Synchronization

| Tool | Suspends? | Use For |
|------|-----------|---------|
| `Mutex` | ✅ Yes | Single-resource exclusive access |
| `Semaphore` | ✅ Yes | Limit number of concurrent operations |
| `synchronized` | ❌ Blocks thread | Non-coroutine Java interop only |
| `AtomicInteger` | N/A | Simple counter without lock |
| `StateFlow.update` | N/A | Atomic state updates |

---

## Atomic Alternatives

```kotlin
// ✅ AtomicInteger for simple counters — no Mutex needed
class RequestCounter {
    private val count = AtomicInteger(0)
    fun increment() = count.incrementAndGet()
    fun get() = count.get()
}

// ✅ StateFlow.update for UI state — thread-safe without Mutex
private val _state = MutableStateFlow(UiState())
fun updateUser(user: User) {
    _state.update { it.copy(user = user) }  // atomic — no Mutex needed
}
```

---

## tryLock — Non-Blocking Attempt

```kotlin
// ✅ tryLock — skip if already locked
class PeriodicSync @Inject constructor() {
    private val syncMutex = Mutex()

    suspend fun trySyncNow() {
        if (!syncMutex.tryLock()) {
            // sync already in progress — skip
            return
        }
        try {
            performSync()
        } finally {
            syncMutex.unlock()
        }
    }
}
```

---

## Anti-Patterns

- Using `synchronized {}` in coroutine code — blocks the thread, not coroutine-friendly
- Long operations inside `withLock` — holds the lock too long, blocks other coroutines
- Nested `withLock` calls on the same Mutex — deadlock
- Using Mutex for simple counter updates — use `AtomicInteger` instead
- Not releasing the lock in finally — use `withLock` extension, not manual lock/unlock

---

## Related Skills
- `coroutine` — coroutine fundamentals
- `structured-concurrency` — parent-child relationships and cancellation
- `synchronization-policy` — broader thread safety strategy
- `authentication` — token refresh with Mutex
