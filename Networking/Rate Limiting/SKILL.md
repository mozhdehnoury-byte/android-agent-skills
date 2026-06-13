---
name: rate-limiting
description: >
  Handling rate limiting (HTTP 429) and implementing client-side request
  throttling in Android. Load this skill when handling 429 Too Many Requests
  responses, respecting Retry-After headers, implementing request queuing,
  or throttling outgoing requests to avoid hitting server limits.
---

# Rate Limiting

## Overview
Rate limiting occurs when the server rejects requests because the client has exceeded the allowed request frequency. On Android, this is handled in two ways: reacting to 429 responses from the server, and proactively throttling client-side requests to avoid hitting the limit.

---

## Core Principles

- Always respect the `Retry-After` header when present — don't guess the delay
- Implement **client-side throttling** for high-frequency operations (search, autocomplete)
- Use a **single retry** for 429 — not an infinite loop
- Log rate limit events — they indicate either a bug or a need to adjust request frequency
- Debounce user-driven requests (search input) before they reach the network

---

## Handling 429 Server Response

```kotlin
// ✅ 429 handler in repository
suspend fun <T> withRateLimitHandling(block: suspend () -> T): Result<T> {
    return runCatching { block() }
        .recoverCatching { throwable ->
            if (throwable is HttpException && throwable.code() == 429) {
                val retryAfterSeconds = throwable.response()
                    ?.headers()
                    ?.get("Retry-After")
                    ?.toLongOrNull()
                    ?: 10L  // default 10 seconds if header missing

                delay(retryAfterSeconds * 1_000)
                block()  // single retry after waiting
            } else {
                throw throwable
            }
        }
}

// ✅ Usage
override suspend fun searchUsers(query: String): Result<List<User>> =
    withRateLimitHandling {
        api.searchUsers(query).map { mapper.toDomain(it) }
    }
```

---

## OkHttp Rate Limit Interceptor

```kotlin
// ✅ Intercept 429 at the HTTP level
class RateLimitInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())

        if (response.code == 429) {
            val retryAfter = response.headers["Retry-After"]?.toLongOrNull() ?: 10L
            response.close()
            Thread.sleep(retryAfter * 1_000)
            return chain.proceed(chain.request())
        }

        return response
    }
}
```

---

## Client-Side Debouncing (Search / Autocomplete)

```kotlin
// ✅ Debounce search input in ViewModel — reduce API calls
class SearchViewModel @Inject constructor(
    private val searchUseCase: SearchUsersUseCase
) : ViewModel() {

    private val _query = MutableStateFlow("")
    private val _results = MutableStateFlow<UiState<List<User>>>(UiState.Loading)
    val results: StateFlow<UiState<List<User>>> = _results.asStateFlow()

    init {
        viewModelScope.launch {
            _query
                .debounce(300)              // ✅ wait 300ms after last keystroke
                .filter { it.length >= 2 }  // ✅ minimum query length
                .distinctUntilChanged()     // ✅ skip duplicate queries
                .collectLatest { query ->   // ✅ cancel previous in-flight request
                    _results.value = UiState.Loading
                    searchUseCase(query).fold(
                        onSuccess = { _results.value = UiState.Success(it) },
                        onFailure = { _results.value = UiState.Error(it.message ?: "Error") }
                    )
                }
        }
    }

    fun onQueryChange(query: String) {
        _query.value = query
    }
}
```

---

## Request Throttler (Token Bucket)

```kotlin
// ✅ Token bucket throttler — max N requests per window
class RequestThrottler(
    private val maxRequests: Int = 10,
    private val windowMs: Long = 1_000L
) {
    private val mutex = Mutex()
    private val timestamps = ArrayDeque<Long>()

    suspend fun throttle() {
        mutex.withLock {
            val now = System.currentTimeMillis()
            // remove timestamps outside the window
            while (timestamps.isNotEmpty() && now - timestamps.first() > windowMs) {
                timestamps.removeFirst()
            }
            if (timestamps.size >= maxRequests) {
                val waitMs = windowMs - (now - timestamps.first())
                delay(waitMs)
            }
            timestamps.addLast(System.currentTimeMillis())
        }
    }
}

// ✅ Usage
class AnalyticsDataSource @Inject constructor(
    private val throttler: RequestThrottler
) {
    suspend fun trackEvent(event: AnalyticsEvent) {
        throttler.throttle()
        api.track(event)
    }
}
```

---

## Anti-Patterns

- Retrying 429 immediately without delay — will get another 429 immediately
- Ignoring the `Retry-After` header — use the server-specified delay
- Not debouncing search input — fires a request on every keystroke
- Infinite retry loop on 429 — cap at one retry after the wait
- Using `Thread.sleep` in coroutine context — use `delay` instead

---

## Related Skills
- `retry-backoff` — general retry strategy for transient failures
- `okhttp` — interceptor setup for HTTP-level handling
- `flow` — debounce and distinctUntilChanged operators
- `fallback-strategy` — what to do when rate limit retries fail
