---
name: workmanager
description: >
  WorkManager for guaranteed background task execution in Android.
  Load this skill when scheduling deferrable work, running periodic tasks,
  chaining workers, handling constraints, observing work state,
  or running tasks that must survive app and device restarts.
---

# WorkManager

## Overview
WorkManager is the recommended solution for deferrable, guaranteed background work in Android. It handles Doze mode, app death, and device restarts transparently. It supports one-time and periodic work, constraints, chaining, and progress reporting.

---

## Core Principles

- Use WorkManager for work that **must complete** even if the app is killed
- Use `CoroutineWorker` — not `Worker` — for suspend-friendly work
- Set **constraints** to avoid running on metered networks or low battery
- Use **unique work** to prevent duplicate tasks
- Observe work state via `WorkInfo` — never poll manually

---

## Setup

```toml
# libs.versions.toml
[versions]
workmanager = "2.9.0"

[libraries]
workmanager = { module = "androidx.work:work-runtime-ktx", version.ref = "workmanager" }
```

---

## CoroutineWorker

```kotlin
// ✅ Standard worker
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            val userId = inputData.getString(KEY_USER_ID)
                ?: return Result.failure()

            syncRepository.syncUser(userId)
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry()
            else Result.failure(
                workDataOf("error" to (e.message ?: "Unknown"))
            )
        }
    }

    companion object {
        const val KEY_USER_ID = "user_id"
    }
}
```

---

## One-Time Work

```kotlin
// ✅ Enqueue one-time work
fun scheduleSyncUser(userId: String) {
    val request = OneTimeWorkRequestBuilder<SyncWorker>()
        .setInputData(workDataOf(SyncWorker.KEY_USER_ID to userId))
        .setConstraints(
            Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build()
        )
        .setBackoffCriteria(
            BackoffPolicy.EXPONENTIAL,
            WorkRequest.MIN_BACKOFF_MILLIS,
            TimeUnit.MILLISECONDS
        )
        .addTag("sync")
        .build()

    WorkManager.getInstance(context).enqueueUniqueWork(
        "sync_user_$userId",
        ExistingWorkPolicy.KEEP,   // don't replace if already queued
        request
    )
}
```

---

## Periodic Work

```kotlin
// ✅ Periodic sync — minimum interval is 15 minutes
fun schedulePeriodicSync() {
    val request = PeriodicWorkRequestBuilder<SyncWorker>(
        repeatInterval = 1,
        repeatIntervalTimeUnit = TimeUnit.HOURS
    )
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
        request
    )
}
```

---

## Chaining Workers

```kotlin
// ✅ Sequential chain
WorkManager.getInstance(context)
    .beginUniqueWork(
        "upload_pipeline",
        ExistingWorkPolicy.REPLACE,
        OneTimeWorkRequestBuilder<CompressWorker>().build()
    )
    .then(OneTimeWorkRequestBuilder<UploadWorker>().build())
    .then(OneTimeWorkRequestBuilder<NotifyWorker>().build())
    .enqueue()

// ✅ Parallel then merge
val syncA = OneTimeWorkRequestBuilder<SyncAWorker>().build()
val syncB = OneTimeWorkRequestBuilder<SyncBWorker>().build()
val merge = OneTimeWorkRequestBuilder<MergeWorker>().build()

WorkManager.getInstance(context)
    .beginWith(listOf(syncA, syncB))
    .then(merge)
    .enqueue()
```

---

## Progress Reporting

```kotlin
// ✅ Report progress from worker
class UploadWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        val files = getFilesToUpload()
        files.forEachIndexed { index, file ->
            uploadFile(file)
            val progress = ((index + 1) * 100) / files.size
            setProgress(workDataOf("progress" to progress))
        }
        return Result.success()
    }
}

// ✅ Observe progress in ViewModel
val workInfo: Flow<WorkInfo?> = workManager
    .getWorkInfosForUniqueWorkFlow("upload_pipeline")
    .map { it.firstOrNull() }

// In composable
val progress = workInfo?.progress?.getInt("progress", 0) ?: 0
```

---

## Observing Work State

```kotlin
// ✅ Observe in ViewModel
class SyncViewModel @Inject constructor(
    private val workManager: WorkManager
) : ViewModel() {

    val syncState: StateFlow<SyncUiState> = workManager
        .getWorkInfosForUniqueWorkFlow("periodic_sync")
        .map { workInfos ->
            when (workInfos.firstOrNull()?.state) {
                WorkInfo.State.RUNNING   -> SyncUiState.Syncing
                WorkInfo.State.SUCCEEDED -> SyncUiState.Success
                WorkInfo.State.FAILED    -> SyncUiState.Failed
                else                     -> SyncUiState.Idle
            }
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), SyncUiState.Idle)
}
```

---

## Hilt Integration

```kotlin
// ✅ Inject dependencies into Worker with Hilt
@HiltWorker
class SyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val syncRepository: SyncRepository
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            syncRepository.sync()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// In Application class
@HiltAndroidApp
class App : Application(), Configuration.Provider {
    @Inject lateinit var workerFactory: HiltWorkerFactory

    override val workManagerConfiguration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

---

## Anti-Patterns

- Using `Worker` instead of `CoroutineWorker` — blocks the thread
- Not using unique work — creates duplicate background tasks
- Not setting network constraints for network-dependent workers
- Doing UI updates directly inside a Worker — use WorkInfo observation
- Using WorkManager for immediate short tasks — use coroutines instead

---

## Related Skills
- `background-processing` — choosing the right background tool
- `foreground-service` — long-running user-visible work
- `hilt` — injecting dependencies into workers
- `coroutine` — coroutine fundamentals used inside CoroutineWorker
