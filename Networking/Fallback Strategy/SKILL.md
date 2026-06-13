---
name: fallback-strategy
description: >
  Fallback strategies for handling network and data failures in Android.
  Load this skill when deciding what to show after all retries are exhausted,
  serving stale cache on network failure, implementing graceful degradation,
  or building resilient offline-capable features.
---

# Fallback Strategy

## Overview
A fallback strategy defines what the app does when the primary data source fails. Instead of showing an error and stopping, the app degrades gracefully — serving cached data, a default value, or a reduced-functionality state. The goal is to keep the user productive even when connectivity or the server is unavailable.

---

## Core Principles

- Always try the **fastest/most reliable source first** — then fall back in order
- Serve **stale data with a staleness indicator** rather than a hard error when possible
- Fallback order: memory cache → disk cache → network → default/empty
- Never silently serve stale data without indicating it to the user
- Fallback behavior must be **explicit in the repository** — not implicit

---

## Fallback Chain Pattern

```kotlin
// ✅ Explicit fallback chain in repository
class UserRepositoryImpl @Inject constructor(
    private val remoteSource: UserRemoteDataSource,
    private val localSource: UserLocalDataSource,
    private val memoryCache: UserMemoryCache
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> {
        // 1. Memory cache
        memoryCache.get(id)?.let { return Result.success(it) }

        // 2. Network
        val networkResult = runCatching { remoteSource.getUser(id) }
        if (networkResult.isSuccess) {
            val user = networkResult.getOrThrow()
            memoryCache.put(id, user)
            localSource.saveUser(user)
            return Result.success(user)
        }

        // 3. Disk cache fallback
        val cached = localSource.getUser(id)
        if (cached != null) {
            return Result.success(cached.copy(isStale = true))
        }

        // 4. All sources failed
        return networkResult  // propagate original error
    }
}
```

---

## Stale Data Model

```kotlin
// ✅ Domain model carries staleness flag
data class User(
    val id: String,
    val name: String,
    val email: String,
    val isStale: Boolean = false  // true when served from cache after network failure
)

// ✅ UI shows staleness indicator
@Composable
fun UserDetailScreen(state: UserDetailUiState) {
    if (state is UserDetailUiState.Success) {
        if (state.user.isStale) {
            StaleBanner(message = "Showing cached data — pull to refresh")
        }
        UserContent(state.user)
    }
}
```

---

## Network-First with Cache Fallback (Flow)

```kotlin
// ✅ Emit cached data immediately, then update with network
override fun getUserStream(id: String): Flow<Result<User>> = flow {
    // Emit cached immediately
    val cached = localSource.getUser(id)
    if (cached != null) emit(Result.success(cached.copy(isStale = true)))

    // Fetch fresh from network
    runCatching { remoteSource.getUser(id) }
        .onSuccess { fresh ->
            localSource.saveUser(fresh)
            emit(Result.success(fresh))
        }
        .onFailure { error ->
            if (cached == null) emit(Result.failure(error))
            // else already emitted stale — don't emit error
        }
}
```

---

## Default Value Fallback

```kotlin
// ✅ Return sensible defaults when data is unavailable
override suspend fun getAppConfig(): AppConfig {
    return runCatching { remoteSource.getConfig() }
        .getOrElse {
            localSource.getConfig() ?: AppConfig.default()
        }
}

// ✅ Default config
data class AppConfig(
    val featureFlags: Map<String, Boolean> = emptyMap(),
    val maxUploadSize: Long = 10 * 1024 * 1024L  // 10MB
) {
    companion object {
        fun default() = AppConfig()
    }
}
```

---

## Partial Fallback (Feature Degradation)

```kotlin
// ✅ Load what's available — degrade features that can't load
data class DashboardData(
    val user: User,
    val recentOrders: List<Order> = emptyList(),  // empty = orders unavailable
    val notifications: List<Notification> = emptyList(),
    val ordersError: Boolean = false,
    val notificationsError: Boolean = false
)

suspend fun loadDashboard(userId: String): DashboardData {
    val user = userRepository.getUser(userId).getOrThrow()  // required — throw if fails

    val orders = orderRepository.getRecentOrders(userId)
    val notifications = notificationRepository.getUnread(userId)

    return DashboardData(
        user = user,
        recentOrders = orders.getOrDefault(emptyList()),
        notifications = notifications.getOrDefault(emptyList()),
        ordersError = orders.isFailure,
        notificationsError = notifications.isFailure
    )
}
```

---

## Anti-Patterns

- Showing a hard error screen when cached data is available — use stale data instead
- Silently serving stale data without any indicator — user doesn't know data may be old
- Retrying indefinitely before falling back — set a max retry then fall back
- Treating all fallback data the same as fresh data — mark staleness explicitly
- Fallback logic scattered across ViewModel and Repository — keep it in the repository

---

## Related Skills
- `retry-backoff` — exhausting retries before triggering fallback
- `cache-strategy` — cache implementation details
- `offline-first` — building apps that work without connectivity
- `error-handling` — propagating errors when all fallbacks fail
- `loading-strategy` — showing appropriate loading states during fallback
