---
name: failure-strategy
description: >
  Failure response strategy for Android apps.
  Load this skill when deciding how to respond to different error types,
  implementing retry logic, handling session expiry globally,
  designing fallback behavior, or choosing between silent fail and user notification.
---

# Failure Strategy

## Overview
Failure strategy defines what the app does when an error occurs — not just what message to show, but whether to retry, fall back to cached data, redirect to login, or block the user. Different errors require different responses, and some responses (like session expiry handling) should be centralized rather than handled per-screen.

---

## Core Principles

- **Match response to error severity** — silent fail for non-critical, block for auth failures
- **Centralize cross-cutting failures** — session expiry and network availability are app-level concerns
- **Retry with backoff** for transient errors — network timeouts and server errors
- **Fallback to cache** when possible — offline-first is better than blank error screens
- **Let the user decide** on recoverable errors — show retry, not just error message

---

## Failure Response Matrix

| Error | Response |
|---|---|
| `Network.NoConnection` | Show offline banner + cached data |
| `Network.Timeout` | Auto-retry (1-2x), then show retry button |
| `Network.ServerError` | Show error + retry button |
| `Network.Unauthorized` | Redirect to login (global handler) |
| `Network.Forbidden` | Show permission error, no retry |
| `Network.NotFound` | Show not-found state, no retry |
| `Data.Validation` | Show inline field errors |
| `Data.Conflict` | Show specific conflict message |
| `Storage.DiskFull` | Show disk full warning, prompt to free space |
| `Auth.AccountSuspended` | Show suspension message, no retry |
| `Unexpected` | Log + show generic error + retry |

---

## Retry with Exponential Backoff

```kotlin
// ✅ Retry helper for transient errors
suspend fun <T> retryWithBackoff(
    times: Int = 3,
    initialDelay: Long = 500L,
    maxDelay: Long = 5_000L,
    factor: Double = 2.0,
    retryIf: (Throwable) -> Boolean = { true },
    block: suspend () -> T
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
        try {
            return block()
        } catch (e: CancellationException) {
            throw e  // never retry cancellation
        } catch (e: Exception) {
            if (!retryIf(e)) throw e
            Timber.w(e, "Attempt ${attempt + 1} failed, retrying in ${currentDelay}ms")
            delay(currentDelay)
            currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
        }
    }
    return block()  // last attempt — let it throw
}

// ✅ Usage in Repository
override suspend fun syncData(): Result<Unit> = runCatching {
    retryWithBackoff(
        times = 3,
        retryIf = { e -> e is IOException || (e is HttpException && e.code() >= 500) }
    ) {
        api.syncData()
    }
}
```

---

## Global Session Expiry Handler

```kotlin
// ✅ Intercept 401 globally via OkHttp Authenticator
class TokenRefreshAuthenticator @Inject constructor(
    private val securePreferences: SecurePreferences,
    private val authApi: AuthApiService,
    private val sessionManager: SessionManager
) : Authenticator {

    private val isRefreshing = AtomicBoolean(false)

    override fun authenticate(route: Route?, response: Response): Request? {
        if (response.code != 401) return null
        if (responseCount(response) >= 2) {
            // Refresh failed — force logout
            sessionManager.logout()
            return null
        }

        synchronized(this) {
            if (isRefreshing.get()) return null
            isRefreshing.set(true)

            return try {
                val refreshToken = securePreferences.refreshToken ?: run {
                    sessionManager.logout()
                    return null
                }

                val newTokens = runBlocking { authApi.refreshToken(refreshToken) }
                securePreferences.authToken = newTokens.accessToken
                securePreferences.refreshToken = newTokens.refreshToken

                response.request.newBuilder()
                    .header("Authorization", "Bearer ${newTokens.accessToken}")
                    .build()
            } catch (e: Exception) {
                sessionManager.logout()
                null
            } finally {
                isRefreshing.set(false)
            }
        }
    }

    private fun responseCount(response: Response): Int {
        var count = 1
        var prior = response.priorResponse
        while (prior != null) { count++; prior = prior.priorResponse }
        return count
    }
}

// ✅ SessionManager broadcasts logout event to all screens
class SessionManager @Inject constructor() {
    private val _sessionEvents = MutableSharedFlow<SessionEvent>(extraBufferCapacity = 1)
    val sessionEvents: SharedFlow<SessionEvent> = _sessionEvents.asSharedFlow()

    fun logout() {
        _sessionEvents.tryEmit(SessionEvent.SessionExpired)
    }
}

sealed interface SessionEvent {
    data object SessionExpired : SessionEvent
    data object LoggedOut : SessionEvent
}
```

---

## App-Level Session Observer

```kotlin
// ✅ Observe session events in root ViewModel or Activity
@HiltViewModel
class AppViewModel @Inject constructor(
    private val sessionManager: SessionManager
) : ViewModel() {

    private val _navigationEvents = Channel<AppNavigationEvent>(Channel.BUFFERED)
    val navigationEvents: Flow<AppNavigationEvent> = _navigationEvents.receiveAsFlow()

    init {
        viewModelScope.launch {
            sessionManager.sessionEvents.collect { event ->
                when (event) {
                    SessionEvent.SessionExpired -> {
                        _navigationEvents.send(AppNavigationEvent.NavigateToLogin)
                    }
                    SessionEvent.LoggedOut -> {
                        _navigationEvents.send(AppNavigationEvent.NavigateToLogin)
                    }
                }
            }
        }
    }
}
```

---

## Fallback to Cache Strategy

```kotlin
// ✅ Show stale cache while fetching fresh data
class ProductRepositoryImpl @Inject constructor(
    private val api: ProductApiService,
    private val dao: ProductDao
) : ProductRepository {

    override fun observeProducts(): Flow<List<Product>> = flow {
        // 1. Emit cached data immediately
        val cached = dao.getAll().map { it.toDomain() }
        if (cached.isNotEmpty()) emit(cached)

        // 2. Fetch fresh data in background
        try {
            val fresh = api.getProducts().map { it.toDomain() }
            dao.replaceAll(fresh.map { it.toEntity() })
            emit(fresh)
        } catch (e: IOException) {
            // Network failed — cached data already emitted, just log
            Timber.w(e, "Failed to refresh products, showing cached data")
        }
    }
}
```

---

## Per-Screen Retry Pattern

```kotlin
// ✅ Retry button in UiState
data class ProductListUiState(
    val isLoading: Boolean = false,
    val products: List<Product> = emptyList(),
    val error: ErrorState? = null
)

data class ErrorState(
    val message: String,
    val canRetry: Boolean
)

// ✅ Retry composable
@Composable
fun ErrorView(
    error: ErrorState,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(
        modifier = modifier,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text(error.message)
        if (error.canRetry) {
            Spacer(Modifier.height(16.dp))
            Button(onClick = onRetry) { Text("Try Again") }
        }
    }
}

// ✅ ViewModel
fun loadProducts() {
    viewModelScope.launch {
        _state.update { it.copy(isLoading = true, error = null) }
        getProductsUseCase().fold(
            onSuccess = { products ->
                _state.update { it.copy(isLoading = false, products = products) }
            },
            onFailure = { error ->
                _state.update {
                    it.copy(
                        isLoading = false,
                        error = ErrorState(
                            message = error.toUserMessage(),
                            canRetry = (error as? AppException)?.error?.isRetryable() ?: true
                        )
                    )
                }
            }
        )
    }
}
```

---

## Anti-Patterns

- Same error response for all error types — `Unauthorized` needs redirect, `Timeout` needs retry
- Retrying non-retryable errors — `403 Forbidden` won't succeed on retry; don't offer it
- Handling session expiry in every screen — centralize in OkHttp Authenticator + AppViewModel
- Silently swallowing errors without logging — bugs become invisible
- Infinite retry without backoff — hammers the server during outages

---

## Related Skills
- `error-handling` — error propagation across layers
- `domain-error-model` — typed errors that drive strategy decisions
- `error-mapping` — mapping raw exceptions to domain errors
- `user-friendly-errors` — displaying error states in UI
- `offline-first` — cache fallback as part of failure strategy
- `secure-networking` — token refresh interceptor
