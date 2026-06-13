---
name: datastore
description: >
  Jetpack DataStore (Preferences) setup and usage for Android.
  Load this skill when storing simple key-value settings, user preferences,
  feature flags, or any small structured data that doesn't need a full database.
---

# DataStore (Preferences)

## Overview
Jetpack DataStore is the modern replacement for SharedPreferences. It stores key-value pairs (Preferences DataStore) or typed objects (Proto DataStore) asynchronously using coroutines and Flow. It is safe to call from the main thread and handles concurrent access correctly.

---

## Core Principles

- Use DataStore for **small, simple data** — user settings, session state, feature flags
- Use Room for **structured, queryable data** — DataStore is not a database
- **Never block** to read DataStore — always collect as Flow
- Create **one DataStore instance** per file — never multiple instances for the same file
- Inject DataStore via **Hilt** — never create it in a ViewModel or Repository directly

---

## Setup

```toml
# libs.versions.toml
[versions]
datastore = "1.1.1"

[libraries]
datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastore" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.datastore.preferences)
}
```

---

## Creating DataStore

```kotlin
// ✅ Create via extension — one instance per file
val Context.userPreferencesDataStore: DataStore<Preferences> by preferencesDataStore(
    name = "user_preferences"
)

// ✅ Provide via Hilt — singleton
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {

    @Provides
    @Singleton
    fun provideUserPreferencesDataStore(
        @ApplicationContext context: Context
    ): DataStore<Preferences> = context.userPreferencesDataStore
}
```

---

## Defining Keys

```kotlin
// ✅ Define all keys in a companion object or object
object UserPreferencesKeys {
    val IS_DARK_MODE = booleanPreferencesKey("is_dark_mode")
    val LANGUAGE = stringPreferencesKey("language")
    val FONT_SIZE = intPreferencesKey("font_size")
    val LAST_SYNC = longPreferencesKey("last_sync_timestamp")
    val ONBOARDING_COMPLETE = booleanPreferencesKey("onboarding_complete")
}
```

---

## Repository Pattern

```kotlin
// ✅ Wrap DataStore in a Repository — don't access it directly from ViewModel
class UserPreferencesRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {

    // ✅ Read — expose as Flow
    val isDarkMode: Flow<Boolean> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs -> prefs[UserPreferencesKeys.IS_DARK_MODE] ?: false }

    val language: Flow<String> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs -> prefs[UserPreferencesKeys.LANGUAGE] ?: "en" }

    // ✅ Read all at once — single object
    val userPreferences: Flow<UserPreferences> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            UserPreferences(
                isDarkMode = prefs[UserPreferencesKeys.IS_DARK_MODE] ?: false,
                language = prefs[UserPreferencesKeys.LANGUAGE] ?: "en",
                fontSize = prefs[UserPreferencesKeys.FONT_SIZE] ?: 14
            )
        }

    // ✅ Write — suspend function
    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.edit { prefs ->
            prefs[UserPreferencesKeys.IS_DARK_MODE] = enabled
        }
    }

    suspend fun setLanguage(language: String) {
        dataStore.edit { prefs ->
            prefs[UserPreferencesKeys.LANGUAGE] = language
        }
    }

    // ✅ Clear all preferences
    suspend fun clearAll() {
        dataStore.edit { it.clear() }
    }
}

// ✅ Domain model for preferences
data class UserPreferences(
    val isDarkMode: Boolean,
    val language: String,
    val fontSize: Int
)
```

---

## Usage in ViewModel

```kotlin
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val preferencesRepository: UserPreferencesRepository
) : ViewModel() {

    val preferences: StateFlow<UserPreferences> = preferencesRepository.userPreferences
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UserPreferences(isDarkMode = false, language = "en", fontSize = 14)
        )

    fun onDarkModeToggled(enabled: Boolean) {
        viewModelScope.launch {
            preferencesRepository.setDarkMode(enabled)
        }
    }

    fun onLanguageSelected(language: String) {
        viewModelScope.launch {
            preferencesRepository.setLanguage(language)
        }
    }
}
```

---

## Atomic Updates

```kotlin
// ✅ update multiple keys atomically in one edit block
suspend fun saveUserSession(userId: String, token: String, expiresAt: Long) {
    dataStore.edit { prefs ->
        prefs[USER_ID_KEY] = userId
        prefs[TOKEN_KEY] = token
        prefs[EXPIRES_AT_KEY] = expiresAt
    }
}

// ✅ Conditional update
suspend fun incrementLaunchCount() {
    dataStore.edit { prefs ->
        val current = prefs[LAUNCH_COUNT_KEY] ?: 0
        prefs[LAUNCH_COUNT_KEY] = current + 1
    }
}
```

---

## Error Handling

```kotlin
// ✅ Always handle IOException — disk read/write can fail
val preferences: Flow<AppPreferences> = dataStore.data
    .catch { exception ->
        if (exception is IOException) {
            // Emit defaults on read error
            emit(emptyPreferences())
        } else {
            throw exception  // re-throw unexpected errors
        }
    }
    .map { prefs -> prefs.toAppPreferences() }
```

---

## DataStore vs SharedPreferences

| | DataStore | SharedPreferences |
|--|-----------|------------------|
| Thread safety | ✅ Coroutines | ❌ Partial |
| Main thread safe | ✅ Yes | ❌ No (apply is async, commit blocks) |
| Strongly typed | ✅ Yes | ❌ No |
| Error handling | ✅ Flow catch | ❌ Silent |
| Transactions | ✅ edit {} | ❌ No |
| Migration | ✅ SharedPreferencesMigration | — |

---

## Migrating from SharedPreferences

```kotlin
// ✅ Migrate existing SharedPreferences data
val dataStore = context.createDataStore(
    name = "user_preferences",
    migrations = listOf(
        SharedPreferencesMigration(context, "legacy_prefs")
    )
)
```

---

## Anti-Patterns

- Multiple DataStore instances for same file — data corruption risk
- Calling `dataStore.data.first()` in a loop — use Flow collection instead
- Storing large objects or lists in DataStore — use Room instead
- Not catching `IOException` — unhandled disk errors crash the app
- Accessing DataStore directly in ViewModel — wrap in Repository
- Using DataStore for data that needs querying/filtering — use Room

---

## Related Skills
- `proto-datastore` — typed object storage with Protobuf
- `repository-pattern` — wrapping DataStore in a Repository
- `hilt` — providing DataStore as a singleton
- `key-value-store-strategy` — choosing the right storage for the use case
