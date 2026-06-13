---
name: key-value-store-strategy
description: >
  Choosing the right key-value storage for Android — DataStore, SharedPreferences,
  EncryptedSharedPreferences, or in-memory. Load this skill when deciding where
  to store simple values, flags, tokens, or settings.
---

# Key-Value Store Strategy

## Overview

Android provides several key-value storage options. Choosing the wrong one leads to security issues, data loss, or unnecessary complexity. This skill defines which store to use for each type of data.

---

## Decision Matrix

| Data Type                      | Storage                         | Reason                             |
| ------------------------------ | ------------------------------- | ---------------------------------- |
| User settings, preferences     | DataStore (Preferences)         | Async, type-safe, coroutine-native |
| Structured settings object     | DataStore (Proto)               | Schema-safe, evolves cleanly       |
| Auth token, sensitive value    | EncryptedSharedPreferences      | Encrypted at rest                  |
| Session flags (in-memory only) | In-memory (ViewModel/singleton) | No persistence needed              |
| Feature flags from server      | DataStore + remote config       | Persisted, refreshable             |
| Legacy migration source        | SharedPreferences → DataStore   | Migrate away from SP               |

---

## DataStore (Preferences) — Default Choice

```kotlin
// ✅ Use for: app settings, UI preferences, onboarding state, feature flags
val IS_ONBOARDING_DONE = booleanPreferencesKey("onboarding_done")
val SELECTED_LANGUAGE  = stringPreferencesKey("language")
val LAST_SYNC_TIME     = longPreferencesKey("last_sync")

// Read
val isDone: Flow<Boolean> = dataStore.data.map { it[IS_ONBOARDING_DONE] ?: false }

// Write
suspend fun markOnboardingDone() {
    dataStore.edit { it[IS_ONBOARDING_DONE] = true }
}
```

---

## EncryptedSharedPreferences — Sensitive Data

```kotlin
// ✅ Use for: auth tokens, refresh tokens, PII, secrets
// Never store tokens in plain DataStore or SharedPreferences

dependencies {
    implementation(libs.security.crypto)
}

fun createEncryptedPrefs(context: Context): SharedPreferences {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    return EncryptedSharedPreferences.create(
        context,
        "secure_prefs",
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
}

// ✅ Provide via Hilt
@Provides
@Singleton
fun provideEncryptedPrefs(@ApplicationContext context: Context): SharedPreferences =
    createEncryptedPrefs(context)

// ✅ Repository wrapping encrypted prefs
class TokenRepository @Inject constructor(
    private val encryptedPrefs: SharedPreferences
) {
    companion object {
        private const val KEY_ACCESS_TOKEN = "access_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
    }

    fun getAccessToken(): String? = encryptedPrefs.getString(KEY_ACCESS_TOKEN, null)
    fun getRefreshToken(): String? = encryptedPrefs.getString(KEY_REFRESH_TOKEN, null)

    fun saveTokens(accessToken: String, refreshToken: String) {
        encryptedPrefs.edit()
            .putString(KEY_ACCESS_TOKEN, accessToken)
            .putString(KEY_REFRESH_TOKEN, refreshToken)
            .apply()
    }

    fun clearTokens() {
        encryptedPrefs.edit()
            .remove(KEY_ACCESS_TOKEN)
            .remove(KEY_REFRESH_TOKEN)
            .apply()
    }
}
```

---

## In-Memory Store — Session-Only Data

```kotlin
// ✅ Use for: data that only lives for the current session
// Examples: current user object after login, selected items, temp state

// ✅ In ViewModel — scoped to screen
@HiltViewModel
class CheckoutViewModel @Inject constructor() : ViewModel() {
    private val _selectedItems = MutableStateFlow<List<Item>>(emptyList())
    val selectedItems = _selectedItems.asStateFlow()
}

// ✅ In singleton — scoped to app session
@Singleton
class SessionStore @Inject constructor() {
    private var _currentUser: User? = null
    val currentUser: User? get() = _currentUser

    fun setUser(user: User) { _currentUser = user }
    fun clearUser() { _currentUser = null }
}
```

---

## SharedPreferences — Legacy Only

```kotlin
// ❌ Don't use in new code — use DataStore instead
// ✅ Only acceptable when:
//   - Migrating existing code incrementally
//   - A third-party library requires SharedPreferences

// ✅ If you must use it, apply() not commit()
prefs.edit().putString("key", "value").apply()  // async
// NOT:
prefs.edit().putString("key", "value").commit()  // blocks main thread
```

---

## Migration: SharedPreferences → DataStore

```kotlin
// ✅ Migrate automatically on first DataStore access
val dataStore = context.createDataStore(
    name = "app_prefs",
    migrations = listOf(
        SharedPreferencesMigration(
            context = context,
            sharedPreferencesName = "legacy_prefs",
            keysToMigrate = setOf("language", "is_dark_mode")  // migrate only needed keys
        )
    )
)
```

---

## Anti-Patterns

- Storing auth tokens in plain DataStore or SharedPreferences — readable without encryption
- Using SharedPreferences in new code — use DataStore instead
- Storing large objects as JSON strings in key-value store — use Room
- Multiple instances of DataStore for the same file — data corruption
- Using `commit()` on SharedPreferences — blocks main thread
- Storing user PII (name, email) in plain storage — security and compliance risk

---

## Related Skills

- `datastore` — Preferences DataStore implementation
- `proto-datastore` — typed object storage
- `encrypted-storage` — Android Keystore and EncryptedSharedPreferences
- `cache-strategy` — caching with key-value stores
