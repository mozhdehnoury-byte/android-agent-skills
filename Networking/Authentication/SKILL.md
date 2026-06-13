---
name: authentication
description: >
  Authentication and token management for Android apps.
  Load this skill when implementing login/logout flows, managing access
  and refresh tokens, handling token expiry, securing token storage,
  or wiring auth state to navigation.
---

# Authentication

## Overview
Authentication in Android involves obtaining tokens (JWT or session), storing them securely, attaching them to requests, and refreshing them before expiry. The auth state drives navigation — unauthenticated users go to the login flow, authenticated users go to the main flow.

---

## Core Principles

- Store tokens in **EncryptedSharedPreferences** — never plain SharedPreferences or DataStore
- Refresh the access token **proactively** before it expires — not reactively after a 401
- Handle 401 responses with a **single refresh attempt** — avoid refresh loops
- Auth state is a `StateFlow` in a singleton — all consumers observe the same source
- Logout clears all tokens and navigates to the auth graph

---

## Token Storage

```kotlin
// ✅ Encrypted token storage
class TokenStorage @Inject constructor(
    @ApplicationContext context: Context
) {
    private val prefs = EncryptedSharedPreferences.create(
        context,
        "auth_prefs",
        MasterKey.Builder(context).setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build(),
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )

    fun saveTokens(accessToken: String, refreshToken: String) {
        prefs.edit()
            .putString(KEY_ACCESS_TOKEN, accessToken)
            .putString(KEY_REFRESH_TOKEN, refreshToken)
            .apply()
    }

    fun getAccessToken(): String? = prefs.getString(KEY_ACCESS_TOKEN, null)
    fun getRefreshToken(): String? = prefs.getString(KEY_REFRESH_TOKEN, null)

    fun clearTokens() {
        prefs.edit().clear().apply()
    }

    companion object {
        private const val KEY_ACCESS_TOKEN = "access_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
    }
}
```

---

## Auth State

```kotlin
// ✅ Auth state as singleton StateFlow
sealed interface AuthState {
    data object Loading : AuthState
    data object Unauthenticated : AuthState
    data class Authenticated(val userId: String) : AuthState
}

@Singleton
class AuthManager @Inject constructor(
    private val tokenStorage: TokenStorage
) {
    private val _state = MutableStateFlow<AuthState>(AuthState.Loading)
    val state: StateFlow<AuthState> = _state.asStateFlow()

    init {
        _state.value = if (tokenStorage.getAccessToken() != null)
            AuthState.Authenticated(getUserIdFromToken())
        else
            AuthState.Unauthenticated
    }

    fun onLoginSuccess(accessToken: String, refreshToken: String, userId: String) {
        tokenStorage.saveTokens(accessToken, refreshToken)
        _state.value = AuthState.Authenticated(userId)
    }

    fun logout() {
        tokenStorage.clearTokens()
        _state.value = AuthState.Unauthenticated
    }

    private fun getUserIdFromToken(): String {
        // decode JWT or read from storage
        return tokenStorage.getAccessToken()?.let { JwtDecoder.getUserId(it) } ?: ""
    }
}
```

---

## Token Refresh Interceptor

```kotlin
// ✅ Single refresh attempt on 401
class TokenRefreshInterceptor @Inject constructor(
    private val tokenStorage: TokenStorage,
    private val authApi: AuthApi,
    private val authManager: AuthManager
) : Interceptor {

    private val refreshLock = Mutex()

    override fun intercept(chain: Interceptor.Chain): Response {
        val response = chain.proceed(chain.request())

        if (response.code != 401) return response

        response.close()

        val refreshed = runBlocking {
            refreshLock.withLock {
                // check if another thread already refreshed
                val currentToken = tokenStorage.getAccessToken()
                val requestToken = chain.request().header("Authorization")
                    ?.removePrefix("Bearer ")

                if (currentToken != requestToken) {
                    // already refreshed by another call — use new token
                    return@withLock true
                }

                runCatching {
                    val newTokens = authApi.refreshToken(
                        tokenStorage.getRefreshToken() ?: return@withLock false
                    )
                    tokenStorage.saveTokens(newTokens.accessToken, newTokens.refreshToken)
                    true
                }.getOrElse {
                    authManager.logout()
                    false
                }
            }
        }

        return if (refreshed) {
            val newRequest = chain.request().newBuilder()
                .header("Authorization", "Bearer ${tokenStorage.getAccessToken()}")
                .build()
            chain.proceed(newRequest)
        } else {
            chain.proceed(chain.request())
        }
    }
}
```

---

## Auth-Gated Navigation

```kotlin
// ✅ Root NavHost reacts to auth state
@Composable
fun AppNavHost(authManager: AuthManager) {
    val authState by authManager.state.collectAsStateWithLifecycle()
    val navController = rememberNavController()

    LaunchedEffect(authState) {
        when (authState) {
            is AuthState.Authenticated -> navController.navigate(MainGraph) {
                popUpTo(AuthGraph) { inclusive = true }
            }
            is AuthState.Unauthenticated -> navController.navigate(AuthGraph) {
                popUpTo(MainGraph) { inclusive = true }
            }
            is AuthState.Loading -> Unit
        }
    }

    NavHost(navController = navController, startDestination = AuthGraph) {
        navigation<AuthGraph>(startDestination = LoginRoute) { /* ... */ }
        navigation<MainGraph>(startDestination = HomeRoute) { /* ... */ }
    }
}
```

---

## Anti-Patterns

- Storing tokens in plain `SharedPreferences` or `DataStore` — not encrypted
- Refreshing token on every request — use proactive refresh or single-retry pattern
- Multiple simultaneous refresh calls — use `Mutex` to serialize refresh
- Keeping auth state in a ViewModel — use a singleton `AuthManager`
- Not clearing tokens on logout — stale tokens remain accessible

---

## Related Skills
- `okhttp` — interceptor setup for auth headers
- `encrypted-storage` — secure storage for sensitive data
- `nested-navigation` — auth vs main flow navigation graphs
- `savedstatehandle` — preserving state across auth transitions
