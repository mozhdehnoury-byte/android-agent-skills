---
name: background-processing
description: >
  Background processing strategies in Android.
  Load this skill when choosing between WorkManager, coroutines, and
  foreground services for background work, scheduling deferred tasks,
  handling doze mode, or running work that must survive app death.
---

# Background Processing

## Overview

Android restricts background processing to preserve battery and memory. The correct tool depends on whether the work must survive app death, needs to be scheduled, or must run immediately. WorkManager is the standard for deferrable, guaranteed work. Coroutines handle in-process background work. Foreground Services handle long-running user-visible work.

---

## Core Principles

- Use **WorkManager** for deferrable work that must survive app death or device restart
- Use **coroutines** (`Dispatchers.IO`) for in-process background work tied to app lifetime
- Use **ForegroundService** for long-running work that the user is aware of (music, upload)
- Never use `AsyncTask`, `HandlerThread`, or bare `Thread` — use coroutines
- Respect **Doze mode** — only WorkManager and ForegroundService work reliably in Doze

---

## Decision Matrix

| Work Type                 | Survives App Death | User Visible | Use                        |
| ------------------------- | ------------------ | ------------ | -------------------------- |
| Short async (network, DB) | ❌                  | ❌            | Coroutine                  |
| Deferred, guaranteed      | ✅                  | ❌            | WorkManager                |
| Periodic background sync  | ✅                  | ❌            | WorkManager                |
| Long-running, user-aware  | ✅                  | ✅            | ForegroundService          |
| Immediate, in-process     | ❌                  | ❌            | Coroutine + Dispatchers.IO |

---

## WorkManager — Deferrable Work

```kotlin
// ✅ Define a Worker
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val userId = inputData.getString("user_id") ?: return Result.failure()
            syncRepository.syncUser(userId)
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()
        }
    }
}

// ✅ Schedule one-time work
val request = OneTimeWorkRequestBuilder<SyncWorker>()
    .setInputData(workDataOf("user_id" to userId))
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 15, TimeUnit.MINUTES)
    .build()

WorkManager.getInstance(context).enqueueUniqueWork(
    "sync_user_$userId",
    ExistingWorkPolicy.KEEP,
    request
)

// ✅ Schedule periodic work
val periodicRequest = PeriodicWorkRequestBuilder<SyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
    )
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "periodic_sync",
    ExistingPeriodicWorkPolicy.KEEP,
    periodicRequest
)
```

---

## Coroutine Background Work

```kotlin
// ✅ Short background work — in-process only
class DataRepository @Inject constructor(
    private val api: DataApi,
    private val dao: DataDao
) {
    suspend fun refreshData() = withContext(Dispatchers.IO) {
        val data = api.getData()
        dao.insertAll(data)
    }
}

// ✅ Long-running in-process work with custom scope
@Singleton
class PollingManager @Inject constructor(
    private val repository: DataRepository
) {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    fun startPolling() {
        scope.launch {
            while (isActive) {
                runCatching { repository.refreshData() }
                delay(30_000)
            }
        }
    }

    fun stopPolling() = scope.cancel()
}
```

---

## Observing WorkManager Progress

```kotlin
// ✅ Observe work state in ViewModel
class SyncViewModel @Inject constructor(
    private val workManager: WorkManager
) : ViewModel() {

    val syncState: LiveData<WorkInfo> = workManager
        .getWorkInfosForUniqueWorkLiveData("periodic_sync")
        .map { workInfos -> workInfos.firstOrNull() }
        .filterNotNull()

    // Or as Flow
    val syncStateFlow: Flow<WorkInfo?> = workManager
        .getWorkInfosForUniqueWorkFlow("periodic_sync")
        .map { it.firstOrNull() }
}
```

---

## Chaining Work

```kotlin
// ✅ Chain workers sequentially
WorkManager.getInstance(context)
    .beginUniqueWork("upload_chain", ExistingWorkPolicy.REPLACE,
        OneTimeWorkRequestBuilder<CompressWorker>().build()
    )
    .then(OneTimeWorkRequestBuilder<UploadWorker>().build())
    .then(OneTimeWorkRequestBuilder<NotifyWorker>().build())
    .enqueue()
```

---

## Anti-Patterns

- Using `GlobalScope` for work that should survive — use WorkManager instead
- Running network calls in a bare `Thread` — use coroutines with Dispatchers.IO
- Using `AlarmManager` for periodic work — WorkManager handles this better
- Starting a ForegroundService for short work — use WorkManager
- Not setting constraints on WorkManager — work runs even on metered network or low battery

---

## Related Skills

- `workmanager` — detailed WorkManager patterns
- `foreground-service` — long-running user-visible work
- `coroutine` — coroutine fundamentals and dispatcher selection
- `structured-concurrency` — managing coroutine lifecycle
