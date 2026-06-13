---
name: okhttp
description: >
  OkHttp client configuration and interceptor patterns for Android.
  Load this skill when configuring OkHttp, writing custom interceptors,
  setting up caching, managing timeouts, handling connection pooling,
  or debugging HTTP traffic.
---

# OkHttp

## Overview
OkHttp is the HTTP engine underneath Retrofit. It manages connection pooling, caching, interceptors, and TLS configuration. Correct OkHttp setup is critical for performance, security, and reliability.

---

## Core Principles

- One `OkHttpClient` instance shared across all Retrofit instances — it manages the connection pool
- Use **interceptors** for cross-cutting concerns: auth, logging, retry, headers
- Configure **timeouts** explicitly — never rely on defaults in production
- Use `Cache` for GET responses to reduce bandwidth and improve offline resilience
- Never block the OkHttp thread inside an interceptor — use coroutines at the repository level

---

## Client Setup

```kotlin
// ✅ Full OkHttpClient configuration
@Provides
@Singleton
fun provideOkHttpClient(
    authInterceptor: AuthInterceptor,
    loggingInterceptor: HttpLoggingInterceptor,
    cache: Cache
): OkHttpClient {
    return OkHttpClient.Builder()
        // Interceptors (order matters — application interceptors first)
        .addInterceptor(authInterceptor)
        .addInterceptor(loggingInterceptor)
        // Timeouts
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .callTimeout(60, TimeUnit.SECONDS)
        // Cache
        .cache(cache)
        // Connection pool
        .connectionPool(ConnectionPool(5, 5, TimeUnit.MINUTES))
        .build()
}

@Provides
@Singleton
fun provideCache(@ApplicationContext context: Context): Cache {
    val cacheSize = 10L * 1024 * 1024  // 10 MB
    return Cache(context.cacheDir, cacheSize)
}
```

---

## Interceptor Types

```
Application Interceptors (addInterceptor)
  → Run before the request hits the network
  → See the original request and final response
  → Good for: auth headers, logging, retry

Network Interceptors (addNetworkInterceptor)
  → Run after redirects and cache decisions
  → See the actual network request/response
  → Good for: cache control headers, compression
```

---

## Custom Interceptors

```kotlin
// ✅ Header interceptor — add common headers to every request
class CommonHeadersInterceptor @Inject constructor(
    private val appVersionProvider: AppVersionProvider
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request().newBuilder()
            .addHeader("X-App-Version", appVersionProvider.version)
            .addHeader("X-Platform", "android")
            .addHeader("Accept-Language", Locale.getDefault().language)
            .build()
        return chain.proceed(request)
    }
}

// ✅ Token refresh interceptor — handle 401 by refreshing token
class TokenRefreshInterceptor @Inject constructor(
    private val tokenRepository: TokenRepository
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())

        if (response.code == 401) {
            response.close()
            val newToken = runBlocking { tokenRepository.refreshToken() }
            if (newToken != null) {
                val newRequest = chain.request().newBuilder()
                    .header("Authorization", "Bearer $newToken")
                    .build()
                return chain.proceed(newRequest)
            }
        }
        return response
    }
}

// ✅ Cache control interceptor — force cache for specific endpoints
class CacheControlInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())
        return if (chain.request().url.pathSegments.contains("static")) {
            response.newBuilder()
                .header("Cache-Control", "public, max-age=86400")  // 1 day
                .build()
        } else {
            response
        }
    }
}
```

---

## Logging Interceptor

```kotlin
// ✅ Debug-only logging — never log in release
@Provides
@Singleton
fun provideLoggingInterceptor(): HttpLoggingInterceptor {
    return HttpLoggingInterceptor { message ->
        Timber.tag("OkHttp").d(message)
    }.apply {
        level = if (BuildConfig.DEBUG)
            HttpLoggingInterceptor.Level.BODY
        else
            HttpLoggingInterceptor.Level.NONE
    }
}
```

---

## Response Extension

```kotlin
// ✅ Safe response body parsing
fun Response.bodyOrThrow(): String {
    return body?.string() ?: throw IOException("Empty response body")
}

fun Response.isSuccess(): Boolean = code in 200..299
```

---

## Timeout Strategy

| Timeout | Recommended | Use Case |
|---------|-------------|----------|
| `connectTimeout` | 15–30s | Establishing TCP connection |
| `readTimeout` | 30–60s | Reading response body |
| `writeTimeout` | 30–60s | Sending request body |
| `callTimeout` | 60–120s | Entire call including redirects |

---

## Anti-Patterns

- Creating a new `OkHttpClient` per request — destroys connection pool benefits
- Using `runBlocking` inside interceptors for non-token-refresh logic — blocks threads
- Logging request bodies in release builds — security risk
- Setting `callTimeout` too low for file upload/download endpoints
- Adding too many interceptors in sequence that each read the response body — body can only be read once

---

## Related Skills
- `retrofit` — Retrofit setup on top of OkHttp
- `authentication` — token management and refresh flow
- `certificate-pinning` — TLS security via OkHttp
- `retry-backoff` — retry interceptor patterns
