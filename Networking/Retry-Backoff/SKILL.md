---
name: retry-backoff
description: >
  Retry and exponential backoff strategies for network requests in Android.
  Load this skill when implementing automatic retry for failed requests,
  configuring backoff intervals, handling idempotency, or building
  resilient network layers.
---

# Retry / Backoff

## Overview
Retry with exponential backoff improves resilience against transient network failures and temporary server errors. The key constraint is that only **idempotent** requests (GET, PUT, DELETE) should be retried automatically — POST requests must not be retried without explicit idempotency tokens.

---

## Core Principles

- Only retry **idempotent** operations automatically — never POST without idempotency token
- Use **exponential backoff** with jitter — prevents thundering herd on server recovery
- Set a **maximum retry count** — don't retry indefinitely
- Do not retry **4xx errors** (except 429) — they indicate a client error, not transient failure
- Retry **5xx errors** and network errors (`IOException`) — these are transient

---

## Retryable Conditions

| Condition | Retry? |
|-----------|--------|
| `IOException` (no internet) | ✅ Yes |
| `500` Server Error | ✅ Yes |
| `502` Bad Gateway | ✅ Yes |
| `503` Service Unavailable | ✅ Yes |
| `429` Too Many Requests | ✅ Yes (after Retry-After) |
| `401` Unauthorized | ❌ No (handle via token refresh) |
| `404` Not Found | ❌ No |
| `400` Bad Request | ❌ No |

---

## Coroutine Retry Extension

```kotlin
// ✅ Generic retry with exponential backoff
suspend fun <T> withRetry(
    maxAttempts: Int = 3,
    initialDelay: Long = 500L,
    maxDelay: Long = 10_000L,
    factor: Double = 2.0,
    shouldRetry: (Throwable) -> Boolean = ::isRetryable,
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(maxAttempts - 1) { attempt ->
        runCatching { block() }
            .onSuccess { return it }
            .onFailure { throwable ->
                if (!shouldRetry(throwable)) throw throwable
                val jitter = (0..200).random()
                delay(currentDelay + jitter)
                currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
            }
    }
    return block()  // last attempt — let it throw
}

fun isRetryable(throwable: Throwable): Boolean {
    return when (throwable) {
        is IOException -> true
        is HttpException -> throwable.code() in listOf(500, 502, 503, 504)
        else -> false
    }
}
```

---

## Usage in Repository

```kotlin
// ✅ Wrap idempotent calls with retry
override suspend fun getUser(id: String): Result<User> = runCatching {
    withRetry(maxAttempts = 3) {
        api.getUser(id)
    }.let { mapper.toDomain(it) }
}

// ✅ Non-idempotent — no automatic retry
override suspend fun createUser(user: User): Result<User> = runCatching {
    val dto = api.createUser(mapper.toRequest(user))  // no withRetry
    mapper.toDomain(dto)
}
```

---

## OkHttp Retry Interceptor

```kotlin
// ✅ Network-level retry for transient failures
class RetryInterceptor(
    private val maxRetries: Int = 3
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()

        // Only retry idempotent methods
        if (request.method !in listOf("GET", "PUT", "DELETE", "HEAD")) {
            return chain.proceed(request)
        }

        var attempt = 0
        var lastException: IOException? = null

        while (attempt < maxRetries) {
            try {
                val response = chain.proceed(request)
                if (response.isSuccessful || response.code !in listOf(500, 502, 503)) {
                    return response
                }
                response.close()
            } catch (e: IOException) {
                lastException = e
            }

            attempt++
            if (attempt < maxRetries) {
                val delay = (500L * 2.0.pow(attempt)).toLong().coerceAtMost(10_000L)
                Thread.sleep(delay)
            }
        }

        throw lastException ?: IOException("Max retries exceeded")
    }
}
```

---

## 429 Too Many Requests Handling

```kotlin
// ✅ Respect Retry-After header on 429
suspend fun <T> withRateLimitRetry(block: suspend () -> T): T {
    while (true) {
        val result = runCatching { block() }
        result.onSuccess { return it }
        result.onFailure { throwable ->
            if (throwable is HttpException && throwable.code() == 429) {
                val retryAfter = throwable.response()
                    ?.headers()
                    ?.get("Retry-After")
                    ?.toLongOrNull()
                    ?: 5L
                delay(retryAfter * 1000)
            } else {
                throw throwable
            }
        }
    }
}
```

---

## Anti-Patterns

- Retrying POST requests without idempotency keys — creates duplicate resources
- Retrying 4xx errors — client errors won't resolve on retry
- Fixed delay between retries — causes thundering herd; use backoff + jitter
- Infinite retries — always cap with `maxAttempts`
- Retrying inside the ViewModel — retry belongs in the repository or data source layer

---

## Related Skills
- `okhttp` — network interceptor for retry at the HTTP level
- `retrofit` — repository-level error handling
- `fallback-strategy` — what to do after all retries are exhausted
- `rate-limiting` — handling 429 responses
