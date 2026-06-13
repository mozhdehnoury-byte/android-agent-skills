---
name: conflict-resolution
description: >
  Conflict resolution strategies for Android offline-first sync.
  Load this skill when local and remote versions of the same data diverge,
  designing conflict detection, or choosing a resolution strategy.
---

# Conflict Resolution

## Overview
A conflict occurs when the same data is modified in both local storage and the remote server before a sync. Conflicts are inevitable in offline-first apps. A conflict resolution strategy defines which version wins — or how to merge them.

---

## Core Principles

- **Detect conflicts explicitly** — don't silently overwrite
- Choose a strategy appropriate to the data type — not all data needs the same policy
- **Last-Write-Wins (LWW)** is simple but lossy — use where data loss is acceptable
- **Server-wins** is safe for critical shared data — client changes are discarded
- **Merge** is complex but preserves both sides — use for collaborative data
- Always log conflicts — they reveal sync bugs and UX pain points

---

## Conflict Detection

```kotlin
// ✅ Track version or timestamp on every entity
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    val email: String,
    @ColumnInfo(name = "updated_at") val updatedAt: Long,      // local modification time
    @ColumnInfo(name = "server_version") val serverVersion: Int, // server's version counter
    @ColumnInfo(name = "sync_state") val syncState: String = "SYNCED"
)

// ✅ Detect conflict: local changed AND server changed since last sync
data class ConflictCandidate(
    val local: UserEntity,
    val remote: UserDto
) {
    val isConflict: Boolean
        get() = local.syncState == "UPDATED" &&
                remote.serverVersion > local.serverVersion
}
```

---

## Strategy 1: Last-Write-Wins (LWW)

```kotlin
// ✅ Simple — most recent timestamp wins
// Use for: user preferences, settings, non-critical data

class LWWConflictResolver {
    fun resolve(local: UserEntity, remote: UserDto): UserEntity {
        return if (remote.updatedAt >= local.updatedAt) {
            // Remote wins — overwrite local
            remote.toEntity()
        } else {
            // Local wins — keep local, mark for push
            local.copy(syncState = "UPDATED")
        }
    }
}
```

---

## Strategy 2: Server-Wins

```kotlin
// ✅ Server is always authoritative
// Use for: financial data, inventory, shared resources, compliance-sensitive data

class ServerWinsConflictResolver {
    fun resolve(local: UserEntity, remote: UserDto): UserEntity {
        // Always use server version — discard local changes
        return remote.toEntity().copy(syncState = "SYNCED")
    }
}
```

---

## Strategy 3: Client-Wins

```kotlin
// ✅ Local changes always take priority — push to server
// Use for: user-authored content, personal notes, drafts

class ClientWinsConflictResolver {
    fun resolve(local: UserEntity, remote: UserDto): UserEntity {
        // Keep local — it will be pushed to server
        return local.copy(
            serverVersion = remote.serverVersion,  // update version tracking
            syncState = "UPDATED"                  // mark for push
        )
    }
}
```

---

## Strategy 4: Field-Level Merge

```kotlin
// ✅ Merge non-overlapping field changes
// Use for: collaborative editing, profile data with independent fields

class FieldMergeConflictResolver {

    fun resolve(base: UserEntity, local: UserEntity, remote: UserDto): UserEntity {
        // Three-way merge: base (last synced) + local changes + remote changes
        return UserEntity(
            id = local.id,
            // If only one side changed a field, take that change
            // If both changed the same field → apply strategy (LWW, server-wins, etc.)
            name = mergeField(
                base = base.name,
                local = local.name,
                remote = remote.name,
                onConflict = { _, remoteVal -> remoteVal }  // server-wins for name
            ),
            email = mergeField(
                base = base.email,
                local = local.email,
                remote = remote.email,
                onConflict = { localVal, _ -> localVal }  // client-wins for email
            ),
            updatedAt = maxOf(local.updatedAt, remote.updatedAt),
            serverVersion = remote.serverVersion,
            syncState = "SYNCED"
        )
    }

    private fun <T> mergeField(
        base: T,
        local: T,
        remote: T,
        onConflict: (T, T) -> T
    ): T {
        val localChanged = local != base
        val remoteChanged = remote != base
        return when {
            localChanged && !remoteChanged -> local
            !localChanged && remoteChanged -> remote
            localChanged && remoteChanged  -> onConflict(local, remote)
            else                           -> base  // neither changed
        }
    }
}
```

---

## Strategy 5: User-Prompted Resolution

```kotlin
// ✅ Let the user decide — for important data where automated resolution is risky

sealed class ConflictResolutionChoice {
    object KeepLocal : ConflictResolutionChoice()
    object KeepRemote : ConflictResolutionChoice()
    data class Merge(val merged: User) : ConflictResolutionChoice()
}

// In ViewModel — emit conflict event
data class ConflictData(val local: User, val remote: User)

sealed class SyncEvent {
    data class ConflictDetected(val conflict: ConflictData) : SyncEvent()
    object SyncComplete : SyncEvent()
}

@HiltViewModel
class SyncViewModel @Inject constructor(
    private val syncEngine: UserSyncEngine
) : ViewModel() {

    private val _events = MutableSharedFlow<SyncEvent>()
    val events: SharedFlow<SyncEvent> = _events.asSharedFlow()

    fun resolveConflict(conflict: ConflictData, choice: ConflictResolutionChoice) {
        viewModelScope.launch {
            syncEngine.applyResolution(conflict, choice)
        }
    }
}
```

---

## Conflict Logging

```kotlin
// ✅ Log every conflict for debugging and analytics
class ConflictLogger @Inject constructor(private val logger: Logger) {

    fun log(local: UserEntity, remote: UserDto, resolution: String) {
        logger.info(
            tag = "ConflictResolution",
            message = buildString {
                append("Conflict for user ${local.id}: ")
                append("local updated_at=${local.updatedAt}, ")
                append("remote updated_at=${remote.updatedAt}, ")
                append("resolution=$resolution")
            }
        )
    }
}
```

---

## Strategy Selection Guide

| Data Type | Recommended Strategy |
|-----------|---------------------|
| User settings / preferences | Last-Write-Wins |
| Financial transactions | Server-wins |
| User-authored content (notes, posts) | Client-wins or User-prompted |
| Shared collaborative data | Field-level merge |
| Inventory / stock levels | Server-wins |
| Profile with independent fields | Field-level merge |

---

## Anti-Patterns

- No conflict detection — silently overwriting data, data loss goes unnoticed
- Always server-wins for user-authored content — users lose their work
- Always client-wins for shared data — other users' changes are lost
- Resolving conflicts on the main thread — always use Dispatchers.IO
- Not logging conflicts — impossible to diagnose sync bugs in production

---

## Related Skills
- `sync-engine` — sync architecture and scheduling
- `merge-strategy` — merging list/collection changes
- `room` — local storage for conflict tracking
- `workmanager` — background sync execution
