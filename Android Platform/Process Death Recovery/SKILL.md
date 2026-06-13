---
name: process-death-recovery
description: >
  Process death detection, prevention, and recovery strategies for Android.
  Load this skill when designing state that must survive system-initiated
  process kills, implementing session recovery, or testing background kill scenarios.
---

# Process Death Recovery

## Overview

Android can kill an app's process at any time when it's in the background — to free memory. When the user returns, Android recreates the Activity and restores the back stack, but all in-memory state (including ViewModel) is gone. A well-designed app recovers seamlessly without data loss.

---

## Core Principles

- Assume process death can happen **at any time** — design accordingly
- ViewModel does **not** survive process death — only configuration change
- SavedStateHandle survives process death — use it for critical UI state
- Persistent storage (Room, DataStore) always survives — use for important data
- Never show a blank or broken screen after process death recovery

---

## What Survives What

| Storage           | Config Change | Process Death | App Kill (swipe) |
| ----------------- | ------------- | ------------- | ---------------- |
| ViewModel         | ✅             | ❌             | ❌                |
| SavedStateHandle  | ✅             | ✅             | ❌                |
| Room / DataStore  | ✅             | ✅             | ✅                |
| SharedPreferences | ✅             | ✅             | ✅                |
| In-memory cache   | ✅             | ❌             | ❌                |

---

## Recovery Strategy by State Type

```kotlin
// ✅ User input — save in SavedStateHandle
@HiltViewModel
class FormViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle
) : ViewModel() {
    val name = savedStateHandle.getStateFlow("name", "")
    val email = savedStateHandle.getStateFlow("email", "")

    fun onNameChanged(value: String) { savedStateHandle["name"] = value }
    fun onEmailChanged(value: String) { savedStateHandle["email"] = value }
}

// ✅ Loaded data — reload from Room/repository on init
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() {
    val users: StateFlow<List<User>> = repository
        .observeUsers()  // Room Flow — always fresh after process death
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}

// ✅ Current screen/selection — save ID in SavedStateHandle, reload data
@HiltViewModel
class UserDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: UserRepository
) : ViewModel() {
    private val userId: String = checkNotNull(savedStateHandle["userId"])

    val user: StateFlow<User?> = repository
        .observeUser(userId)  // reload data using saved ID
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), null)
}
```

---

## Detecting Process Death vs Fresh Start

```kotlin
// ✅ Detect if Activity was restored after process death
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        if (savedInstanceState != null) {
            // Restored — either config change or process death recovery
            // ViewModel + SavedStateHandle already restored
        } else {
            // Fresh start
        }
    }
}

// ✅ In ViewModel — check if state needs initialization
@HiltViewModel
class OnboardingViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
    private val repository: OnboardingRepository
) : ViewModel() {

    private val currentStep = savedStateHandle.getStateFlow("step", 0)

    init {
        // Don't reset step — it may have been restored from SavedStateHandle
        if (currentStep.value == 0) {
            viewModelScope.launch { loadInitialState() }
        }
    }
}
```

---

## Incomplete Operations Recovery

```kotlin
// ✅ For critical operations (payment, upload) — use WorkManager
// WorkManager survives process death and system reboots
class UploadWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val fileUri = inputData.getString("file_uri") ?: return Result.failure()
        return try {
            uploadFile(fileUri)
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// Enqueue — survives process death
WorkManager.getInstance(context)
    .enqueueUniqueWork(
        "upload_${fileId}",
        ExistingWorkPolicy.KEEP,
        OneTimeWorkRequestBuilder<UploadWorker>()
            .setInputData(workDataOf("file_uri" to uri))
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
            .build()
    )
```

---

## Session State Recovery

```kotlin
// ✅ Persist session-critical state to DataStore immediately on change
class SessionRepository @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    suspend fun saveLastViewedUserId(userId: String) {
        dataStore.edit { prefs ->
            prefs[LAST_USER_ID] = userId
        }
    }

    fun observeLastViewedUserId(): Flow<String?> =
        dataStore.data.map { it[LAST_USER_ID] }

    companion object {
        val LAST_USER_ID = stringPreferencesKey("last_user_id")
    }
}
```

---

## Testing Process Death

```bash
# ✅ Simulate process death via ADB — put app in background first
adb shell am kill <package_name>

# Then return to app from Recents — should recover gracefully

# ✅ Android Studio — use "Terminate Application" button
# while app is in background (minimized)
```

```kotlin
// ✅ Write tests for SavedStateHandle restoration
@Test
fun `state is restored after process death`() {
    val savedState = SavedStateHandle(mapOf("search_query" to "kotlin"))
    val viewModel = SearchViewModel(savedState, fakeRepository)

    assertThat(viewModel.query.value).isEqualTo("kotlin")
}
```

---

## Anti-Patterns

- Assuming ViewModel survives process death — it doesn't
- Storing non-Parcelable objects in SavedStateHandle — crashes on restore
- Not testing process death — the most common untested scenario
- Saving too much in SavedStateHandle — it has a size limit (~500KB total)
- Relying on `onSaveInstanceState` in Activity instead of SavedStateHandle in ViewModel
- Starting critical operations (payments, uploads) without WorkManager fallback

---

## Related Skills

- `savedstatehandle` — key-value persistence across process death
- `state-restoration` — full state restoration strategy
- `workmanager` — durable background work across process death
- `datastore` — persisting session state
- `room` — persistent data that always survives
