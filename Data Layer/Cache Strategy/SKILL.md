---
name: cache-strategy
description: >
  Caching patterns and strategies for Android apps — in-memory cache,
  disk cache, network cache, and cache invalidation.
  Load this skill when designing data caching, deciding cache lifetime,
  implementing offline-first patterns, or managing stale data.
---

# Cache Strategy

## Overview

Caching stores copies of data closer to where it's needed to reduce latency, network usage, and battery drain. A good cache strategy balances freshness (up-to-date data) with performance (fast reads). The right strategy depends on how often data changes and how critical freshness is.

---

## Core Principles

- Cache data at the **repository layer** — not in ViewModel or UI
- Define explicit **TTL (time-to-live)** for every cache — stale data is a bug
- **Room is the cache** in offline-first architecture — network fills it, UI reads from it
- Never cache **sensitive data** unencrypted
- Provide a way to **force-refresh** — users expect a pull-to-refresh to work

---

## Cache Strategies

| Strategy                   | When to Use                                                |
| -------------------------- | ---------------------------------------------------------- |
| **Cache-first**            | Offline-first, data changes infrequently                   |
| **Network-first**          | Data must be fresh, connectivity is reliable               |
| **Cache-then-network**     | Show cached data immediately, update when network responds |
| **TTL-based**              | Cache valid for N minutes, refresh after expiry            |
| **Stale-while-revalidate** | Show stale data, fetch fresh in background                 |

---

## Offline-First with Room as Cache

```kotlin
// ✅ Room is the single source of truth — network populates it
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val userApi: UserApi,
    private val userMapper: UserMapper
) {

    // UI always reads from Room — reactive, always fresh
    fun observeUsers(): Flow<List<User>> =
        userDao.observeAll().map { it.map(userMapper::toDomain) }

    // Refresh from network → saves to Room → Room emits new value
    suspend fun refreshUsers(): Result<Unit> = runCatching {
        val remote = userApi.getUsers()
        userDao.upsertAll(remote.map(userMapper::toEntity))
    }
}
```

---

## TTL-Based Cache

```kotlin
// ✅ Track last fetch time — refresh only when stale
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val userApi: UserApi,
    private val prefs: UserPreferencesRepository,
    private val clock: Clock = Clock.systemUTC()
) {
    companion object {
        val CACHE_TTL = Duration.ofMinutes(15)
    }

    fun observeUsers(): Flow<List<User>> = userDao.observeAll()
        .map { it.map(userMapper::toDomain) }

    suspend fun refreshIfStale() {
        val lastSync = prefs.getLastSyncTime()
        val isStale = lastSync == null ||
            Duration.between(lastSync, clock.instant()) > CACHE_TTL

        if (isStale) {
            refreshUsers()
        }
    }

    private suspend fun refreshUsers() {
        val remote = userApi.getUsers()
        userDao.upsertAll(remote.map(userMapper::toEntity))
        prefs.setLastSyncTime(clock.instant())
    }
}
```

---

## In-Memory Cache

```kotlin
// ✅ Simple in-memory LRU cache for expensive computations
class ImageThumbnailCache {
    private val cache = object : LruCache<String, Bitmap>(
        (Runtime.getRuntime().maxMemory() / 1024 / 8).toInt()  // 1/8 of available memory
    ) {
        override fun sizeOf(key: String, value: Bitmap): Int =
            value.byteCount / 1024
    }

    fun get(key: String): Bitmap? = cache.get(key)
    fun put(key: String, bitmap: Bitmap) = cache.put(key, bitmap)
    fun evict(key: String) = cache.remove(key)
    fun clear() = cache.evictAll()
}

// ✅ Simple in-memory cache with TTL
class InMemoryCache<K, V>(private val ttlMs: Long = 5 * 60 * 1000L) {
    private data class Entry<V>(val value: V, val timestamp: Long)
    private val store = ConcurrentHashMap<K, Entry<V>>()

    fun get(key: K): V? {
        val entry = store[key] ?: return null
        return if (System.currentTimeMillis() - entry.timestamp < ttlMs) {
            entry.value
        } else {
            store.remove(key)
            null
        }
    }

    fun put(key: K, value: V) {
        store[key] = Entry(value, System.currentTimeMillis())
    }

    fun invalidate(key: K) = store.remove(key)
    fun invalidateAll() = store.clear()
}
```

---

## Network Cache with OkHttp

```kotlin
// ✅ OkHttp disk cache — caches HTTP responses
val cacheDir = File(context.cacheDir, "http_cache")
val cache = Cache(cacheDir, 10L * 1024 * 1024)  // 10 MB

val okHttpClient = OkHttpClient.Builder()
    .cache(cache)
    .addInterceptor { chain ->
        val request = chain.request().newBuilder()
            .header("Cache-Control", "max-age=300")  // 5 min cache
            .build()
        chain.proceed(request)
    }
    .addNetworkInterceptor { chain ->
        val response = chain.proceed(chain.request())
        response.newBuilder()
            .header("Cache-Control", "public, max-age=300")
            .build()
    }
    .build()
```

---

## Cache Invalidation

```kotlin
// ✅ Invalidate on write — always update cache on mutation
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val userApi: UserApi
) {
    // After creating a user — refresh local cache
    suspend fun createUser(user: User): Result<User> = runCatching {
        val created = userApi.createUser(user.toDto())
        userDao.upsert(created.toEntity())  // cache is updated
        created.toDomain()
    }

    // After deleting — remove from cache
    suspend fun deleteUser(userId: String): Result<Unit> = runCatching {
        userApi.deleteUser(userId)
        userDao.deleteById(userId)  // remove from local cache
    }
}

// ✅ Force refresh — ignore cache
suspend fun forceRefresh() {
    val remote = userApi.getUsers()
    userDao.deleteAll()
    userDao.upsertAll(remote.map { it.toEntity() })
}
```

---

## Pull-to-Refresh Pattern

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {

    val users: StateFlow<List<User>> = repository.observeUsers()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())

    private val _isRefreshing = MutableStateFlow(false)
    val isRefreshing: StateFlow<Boolean> = _isRefreshing.asStateFlow()

    fun onRefresh() {
        viewModelScope.launch {
            _isRefreshing.value = true
            repository.refreshUsers()
            _isRefreshing.value = false
        }
    }
}

// Compose UI
@Composable
fun UserListScreen(viewModel: UserListViewModel = hiltViewModel()) {
    val users by viewModel.users.collectAsStateWithLifecycle()
    val isRefreshing by viewModel.isRefreshing.collectAsStateWithLifecycle()

    SwipeRefresh(
        state = rememberSwipeRefreshState(isRefreshing),
        onRefresh = viewModel::onRefresh
    ) {
        UserList(users = users)
    }
}
```

---

## Anti-Patterns

- Caching in ViewModel — lost on rotation or process death
- No TTL on cache — serves stale data indefinitely
- Caching sensitive data (tokens, PII) in plaintext — use EncryptedSharedPreferences or keystore
- No cache invalidation on write — UI shows stale data after mutations
- Caching everything — large objects in memory cause OOM
- No force-refresh mechanism — users can't recover from stale state

---

## Related Skills

- `repository-pattern` — cache lives in repository layer
- `room` — Room as the offline-first cache
- `datastore` — caching simple values across sessions
- `offline-first` — full offline-first architecture
- `okhttp` — HTTP-level caching
- `key-value-store-strategy` — choosing the right cache storage
