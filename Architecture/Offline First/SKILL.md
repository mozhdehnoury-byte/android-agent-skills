---
name: offline-first
description: >
  Offline-first architecture for Android apps.
  Load this skill when designing apps that must work without network,
  implementing local-first data access, syncing data with a remote server,
  handling network availability changes, or designing cache invalidation strategy.
---

# Offline First

## Overview

Offline-first means the app reads and writes to local storage first. The network is treated as an unreliable sync mechanism — not a requirement. The local database is the single source of truth; the UI observes local data, and a sync engine reconciles local state with the remote server in the background.

---

## Core Principles

- **Local DB is the source of truth** — UI never reads directly from network
- **Write locally first** — operations succeed immediately, sync later
- **Observe local, sync remotely** — UI observes Room/DataStore, background sync updates local DB
- **Surface sync status** — UI shows sync state, errors, last-synced time
- **Idempotent sync** — syncing the same data twice must not corrupt state

---

## Architecture Pattern

```
UI → ViewModel → UseCase → Repository
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              LocalDataSource      RemoteDataSource
              (Room / DataStore)   (Retrofit / Ktor)
                    │
                    ▼
              Single Source of Truth
              (UI observes this)

SyncManager (WorkManager) periodically:
  1. Fetches remote data
  2. Writes to LocalDataSource
  3. UI updates automatically via Flow
```

---

## Repository Implementation

```kotlin
// ✅ Repository — reads local, triggers sync, exposes Flow
class UserRepositoryImpl @Inject constructor(
    private val localDataSource: UserLocalDataSource,
    private val remoteDataSource: UserRemoteDataSource,
    private val syncManager: SyncManager
) : UserRepository {

    // ✅ Always return local Flow — UI updates automatically when sync writes to DB
    override fun observeUsers(): Flow<List<User>> =
        localDataSource.observeAll().map { entities -> entities.map { it.toDomain() } }

    // ✅ Read-through cache
    override suspend fun getUser(id: String): Result<User> = runCatching {
        localDataSource.getById(id)?.toDomain()
            ?: fetchAndCacheUser(id)
    }

    // ✅ Write-through: write locally first, enqueue remote sync
    override suspend fun updateUser(user: User): Result<Unit> = runCatching {
        localDataSource.upsert(user.toEntity().copy(syncStatus = SyncStatus.PENDING))
        syncManager.enqueueUserSync()
    }

    private suspend fun fetchAndCacheUser(id: String): User {
        val dto = remoteDataSource.getUser(id)
        val entity = dto.toDomain().toEntity().copy(syncStatus = SyncStatus.SYNCED)
        localDataSource.upsert(entity)
        return entity.toDomain()
    }
}
```

---

## Sync Status Tracking

```kotlin
// ✅ Track sync state per entity
enum class SyncStatus { SYNCED, PENDING, FAILED }

@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    val syncStatus: SyncStatus = SyncStatus.SYNCED,
    val lastModified: Long = System.currentTimeMillis()
)

// ✅ DAO queries for pending sync
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE syncStatus = 'PENDING'")
    suspend fun getPendingSync(): List<UserEntity>

    @Query("UPDATE users SET syncStatus = :status WHERE id = :id")
    suspend fun updateSyncStatus(id: String, status: SyncStatus)
}
```

---

## Sync Engine with WorkManager

```kotlin
// ✅ Periodic sync via WorkManager
class UserSyncWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted workerParams: WorkerParameters,
    private val userRepository: UserRepository,
    private val remoteDataSource: UserRemoteDataSource,
    private val localDataSource: UserLocalDataSource
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return try {
            // Push pending changes
            val pending = localDataSource.getPendingSync()
            pending.forEach { entity ->
                remoteDataSource.updateUser(entity.toDto())
                localDataSource.updateSyncStatus(entity.id, SyncStatus.SYNCED)
            }

            // Pull remote changes
            val remoteUsers = remoteDataSource.getUsers()
            localDataSource.upsertAll(remoteUsers.map { it.toDomain().toEntity() })

            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }

    companion object {
        fun schedule(workManager: WorkManager) {
            val request = PeriodicWorkRequestBuilder<UserSyncWorker>(15, TimeUnit.MINUTES)
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .build()
                )
                .build()
            workManager.enqueueUniquePeriodicWork(
                "user_sync",
                ExistingPeriodicWorkPolicy.KEEP,
                request
            )
        }
    }
}
```

---

## Network Connectivity Observation

```kotlin
// ✅ Observe network state
class NetworkMonitor @Inject constructor(
    @ApplicationContext private val context: Context
) {
    val isOnline: Flow<Boolean> = callbackFlow {
        val manager = context.getSystemService(ConnectivityManager::class.java)

        val callback = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true) }
            override fun onLost(network: Network) { trySend(false) }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()

        manager.registerNetworkCallback(request, callback)
        trySend(manager.activeNetwork != null)

        awaitClose { manager.unregisterNetworkCallback(callback) }
    }.distinctUntilChanged()
}

// ✅ Surface network state in ViewModel
@HiltViewModel
class HomeViewModel @Inject constructor(
    private val networkMonitor: NetworkMonitor
) : ViewModel() {

    val isOffline: StateFlow<Boolean> = networkMonitor.isOnline
        .map { !it }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), false)
}
```

---

## UI: Offline Banner

```kotlin
// ✅ Show offline indicator when not connected
@Composable
fun OfflineBanner(isOffline: Boolean) {
    AnimatedVisibility(visible = isOffline) {
        Surface(color = MaterialTheme.colorScheme.errorContainer) {
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 16.dp, vertical = 8.dp),
                horizontalArrangement = Arrangement.Center
            ) {
                Icon(Icons.Default.WifiOff, contentDescription = null)
                Spacer(Modifier.width(8.dp))
                Text("You're offline", style = MaterialTheme.typography.labelMedium)
            }
        }
    }
}
```

---

## Anti-Patterns

- Reading directly from network in ViewModel — always read from local source
- No sync status tracking — can't tell which records need to be pushed
- Replacing entire local DB on every sync — causes UI flicker; use upsert
- Syncing on main thread — always use WorkManager or background coroutine
- No conflict resolution strategy — when local and remote diverge, there must be a rule

---

## Related Skills

- `room` — local database for offline storage
- `sync-engine` — advanced sync patterns and conflict resolution
- `workmanager` — background sync scheduling
- `conflict-resolution` — handling remote/local data conflicts
- `repository-pattern` — repository wiring for offline-first
- `cache-strategy` — cache invalidation and expiry
