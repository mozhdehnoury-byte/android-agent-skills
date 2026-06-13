---
name: sync-engine
description: >
  Data synchronization engine design for Android offline-first apps.
  Load this skill when designing how local data syncs with a remote server,
  handling bidirectional sync, managing sync state, or scheduling background sync.
---

# Sync Engine

## Overview
A sync engine coordinates the flow of data between local storage (Room) and a remote server. In offline-first apps, local data is always the source of truth for the UI — the sync engine ensures local and remote stay consistent. It handles connectivity awareness, retry logic, conflict resolution, and sync state tracking.

---

## Core Principles

- **Local first, sync second** — UI reads from Room, sync runs in background
- Sync is **idempotent** — running it twice produces the same result
- Track sync state explicitly — show sync status in UI where relevant
- Use **WorkManager** for background sync — survives process death and reboots
- Handle conflicts explicitly — don't silently overwrite data

---

## Sync Architecture

```
UI → ViewModel → Repository (Room) ← Sync Engine → Remote API
                      ↑
              [Source of Truth]
```

---

## Sync State Tracking

```kotlin
// ✅ Track sync state per entity or globally
enum class SyncStatus {
    IDLE,
    SYNCING,
    SUCCESS,
    FAILED
}

data class SyncState(
    val status: SyncStatus = SyncStatus.IDLE,
    val lastSyncTime: Instant? = null,
    val error: String? = null
)

// ✅ Persist sync metadata
@Entity(tableName = "sync_metadata")
data class SyncMetadataEntity(
    @PrimaryKey val entityType: String,  // "users", "orders", etc.
    val lastSyncTime: Long,
    val lastSyncSuccess: Boolean,
    val pendingChanges: Int = 0
)
```

---

## Sync Engine Implementation

```kotlin
class UserSyncEngine @Inject constructor(
    private val userApi: UserApi,
    private val userDao: UserDao,
    private val syncMetadataDao: SyncMetadataDao,
    private val mapper: UserMapper,
    private val clock: Clock = Clock.systemUTC()
) {
    companion object {
        const val ENTITY_TYPE = "users"
    }

    // ✅ Pull sync — fetch remote, update local
    suspend fun syncFromRemote(): SyncResult {
        return try {
            val lastSync = syncMetadataDao.getLastSyncTime(ENTITY_TYPE)
            val remoteUsers = if (lastSync != null) {
                userApi.getUsersModifiedSince(lastSync)  // delta sync
            } else {
                userApi.getAllUsers()  // full sync on first run
            }

            userDao.upsertAll(remoteUsers.map(mapper::toEntity))

            val deletedIds = userApi.getDeletedSince(lastSync)
            userDao.deleteByIds(deletedIds)

            syncMetadataDao.updateSyncTime(ENTITY_TYPE, clock.instant().toEpochMilli())
            SyncResult.Success(remoteUsers.size)
        } catch (e: IOException) {
            SyncResult.NetworkError(e)
        } catch (e: Exception) {
            SyncResult.UnknownError(e)
        }
    }

    // ✅ Push sync — send local changes to remote
    suspend fun syncToRemote(): SyncResult {
        return try {
            val pendingChanges = userDao.getPendingSync()

            pendingChanges.forEach { entity ->
                when (entity.syncState) {
                    SyncState.CREATED -> userApi.createUser(mapper.toDto(entity))
                    SyncState.UPDATED -> userApi.updateUser(entity.id, mapper.toDto(entity))
                    SyncState.DELETED -> userApi.deleteUser(entity.id)
                }
                userDao.markSynced(entity.id)
            }

            SyncResult.Success(pendingChanges.size)
        } catch (e: Exception) {
            SyncResult.UnknownError(e)
        }
    }
}

sealed class SyncResult {
    data class Success(val count: Int) : SyncResult()
    data class NetworkError(val cause: IOException) : SyncResult()
    data class UnknownError(val cause: Exception) : SyncResult()
}
```

---

## Tracking Local Changes

```kotlin
// ✅ Add sync state column to entity
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    @ColumnInfo(name = "sync_state") val syncState: String = "SYNCED"
)

enum class SyncState { SYNCED, CREATED, UPDATED, DELETED }

// ✅ Mark as pending on local write
@Dao
interface UserDao {
    @Query("UPDATE users SET sync_state = 'UPDATED' WHERE id = :id")
    suspend fun markPendingUpdate(id: String)

    @Query("UPDATE users SET sync_state = 'DELETED' WHERE id = :id")
    suspend fun markPendingDelete(id: String)

    @Query("SELECT * FROM users WHERE sync_state != 'SYNCED'")
    suspend fun getPendingSync(): List<UserEntity>

    @Query("UPDATE users SET sync_state = 'SYNCED' WHERE id = :id")
    suspend fun markSynced(id: String)
}
```

---

## Background Sync with WorkManager

```kotlin
// ✅ Periodic background sync
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    @Inject lateinit var syncEngine: UserSyncEngine

    override suspend fun doWork(): Result {
        return try {
            syncEngine.syncFromRemote()
            syncEngine.syncToRemote()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// ✅ Schedule periodic sync
fun schedulePeriodic(context: Context) {
    WorkManager.getInstance(context).enqueueUniquePeriodicWork(
        "periodic_sync",
        ExistingPeriodicWorkPolicy.KEEP,
        PeriodicWorkRequestBuilder<SyncWorker>(15, TimeUnit.MINUTES)
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 1, TimeUnit.MINUTES)
            .build()
    )
}

// ✅ Trigger immediate sync on app foreground
fun scheduleImmediateSync(context: Context) {
    WorkManager.getInstance(context).enqueueUniqueWork(
        "immediate_sync",
        ExistingWorkPolicy.REPLACE,
        OneTimeWorkRequestBuilder<SyncWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()
    )
}
```

---

## Connectivity-Aware Sync

```kotlin
// ✅ Observe connectivity and trigger sync when online
class ConnectivitySyncTrigger @Inject constructor(
    private val context: Context,
    private val workManager: WorkManager
) {
    fun observe(): Flow<Boolean> = callbackFlow {
        val cm = context.getSystemService(ConnectivityManager::class.java)
        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) {
                trySend(true)
                workManager.enqueueUniqueWork(
                    "connectivity_sync",
                    ExistingWorkPolicy.REPLACE,
                    OneTimeWorkRequestBuilder<SyncWorker>().build()
                )
            }
            override fun onLost(network: Network) { trySend(false) }
        }
        cm.registerDefaultNetworkCallback(callback)
        awaitClose { cm.unregisterNetworkCallback(callback) }
    }
}
```

---

## Anti-Patterns

- Syncing on the main thread — causes ANR
- No retry logic — transient network errors permanently break sync
- Overwriting local changes with remote on every sync — data loss
- No sync state tracking — UI can't show sync status or errors
- Not handling delete propagation — deleted items reappear after sync
- Full sync every time — use delta sync (modified_since) for large datasets

---

## Related Skills
- `conflict-resolution` — handling data conflicts during sync
- `merge-strategy` — merging local and remote changes
- `workmanager` — durable background sync scheduling
- `room` — local data source for offline-first
- `offline-first` — full offline-first architecture
- `repository-pattern` — sync engine integration in repository
