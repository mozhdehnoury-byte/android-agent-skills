---
name: merge-strategy
description: >
  Merge strategies for combining local and remote data changes in Android.
  Load this skill when syncing collections, merging list changes,
  handling additions/deletions on both sides, or implementing CRDT-like patterns.
---

# Merge Strategy

## Overview

A merge strategy defines how to combine two diverged versions of a dataset into a single consistent result. This is more complex than single-record conflict resolution — it handles collections where both sides may have added, modified, and deleted different items.

---

## Core Principles

- Merging requires knowledge of the **base state** — the last known common version
- Track **deletions explicitly** — a missing item could mean "deleted" or "not yet synced"
- Use **stable IDs** as the merge key — never use position or timestamp alone
- Make merge operations **idempotent** — running merge twice yields the same result
- Prefer **additive merges** where possible — deletions are the hardest case

---

## Collection Merge Patterns

### Pattern 1: Remote-Authoritative List Merge

```kotlin
// ✅ Server is source of truth for the list — local additions queued for push
fun mergeUserList(
    localItems: List<UserEntity>,
    remoteItems: List<UserDto>,
    pendingLocalCreates: List<UserEntity>
): List<UserEntity> {
    val remoteById = remoteItems.associateBy { it.id }
    val localById = localItems.associateBy { it.id }

    val merged = mutableListOf<UserEntity>()

    // Add all remote items (authoritative)
    remoteItems.forEach { remote ->
        val local = localById[remote.id]
        merged.add(
            if (local != null && local.updatedAt > remote.updatedAt) {
                local  // local is newer — keep, will push
            } else {
                remote.toEntity()  // remote wins
            }
        )
    }

    // Add local-only creates (not yet on server)
    pendingLocalCreates
        .filter { it.id !in remoteById }
        .forEach { merged.add(it) }

    return merged
}
```

---

### Pattern 2: Three-Way Collection Merge

```kotlin
// ✅ Three-way merge — base + local + remote
data class CollectionDiff<T>(
    val added: List<T>,
    val modified: List<T>,
    val deleted: List<String>  // IDs
)

fun <T : HasId> threeWayMerge(
    base: List<T>,
    local: List<T>,
    remote: List<T>
): List<T> {
    val baseIds = base.map { it.id }.toSet()
    val localIds = local.map { it.id }.toSet()
    val remoteIds = remote.map { it.id }.toSet()

    val baseById   = base.associateBy { it.id }
    val localById  = local.associateBy { it.id }
    val remoteById = remote.associateBy { it.id }

    val result = mutableListOf<T>()

    // Items present in remote
    remoteIds.forEach { id ->
        val remoteItem = remoteById[id]!!
        val localItem  = localById[id]
        val baseItem   = baseById[id]

        when {
            // New on remote — add
            id !in baseIds -> result.add(remoteItem)

            // Deleted locally — don't add (local delete wins)
            id !in localIds -> { /* skip — locally deleted */ }

            // Modified on both sides — resolve conflict
            localItem != baseItem && remoteItem != baseItem ->
                result.add(resolveFieldConflict(localItem!!, remoteItem))

            // Only local changed
            localItem != baseItem -> result.add(localItem!!)

            // Only remote changed, or neither changed
            else -> result.add(remoteItem)
        }
    }

    // Items added locally (not on remote or base)
    localIds
        .filter { it !in baseIds && it !in remoteIds }
        .forEach { result.add(localById[it]!!) }

    return result
}

interface HasId {
    val id: String
}
```

---

### Pattern 3: Additive-Only Merge (No Deletions)

```kotlin
// ✅ Simplest merge — items are never deleted, only added
// Use for: event logs, audit trails, append-only feeds

fun <T : HasId> additiveMerge(
    local: List<T>,
    remote: List<T>
): List<T> {
    val localById = local.associateBy { it.id }
    val remoteById = remote.associateBy { it.id }

    // Union of both sets — remote wins on conflict
    return (localById + remoteById).values.toList()
}
```

---

### Pattern 4: Timestamp-Based List Merge

```kotlin
// ✅ Each item has a timestamp — most recent version of each ID wins
data class TimestampedItem<T>(val id: String, val data: T, val updatedAt: Long)

fun <T> timestampMerge(
    local: List<TimestampedItem<T>>,
    remote: List<TimestampedItem<T>>,
    deletedRemoteIds: Set<String> = emptySet()
): List<TimestampedItem<T>> {
    val merged = (local + remote)
        .groupBy { it.id }
        .mapValues { (_, versions) -> versions.maxByOrNull { it.updatedAt }!! }
        .values
        .filter { it.id !in deletedRemoteIds }
        .toList()

    return merged
}
```

---

## Tracking Deletions

```kotlin
// ✅ Server must provide a tombstone/deleted list
// Without this, you can't distinguish "deleted" from "not synced yet"

@Serializable
data class SyncResponseDto(
    val updated: List<UserDto>,
    val deletedIds: List<String>,  // explicit deletions
    val syncToken: String          // cursor for next delta sync
)

// ✅ Apply deletions explicitly
suspend fun applySync(response: SyncResponseDto) {
    db.withTransaction {
        // Apply updates/additions
        userDao.upsertAll(response.updated.map { it.toEntity() })
        // Apply deletions
        userDao.deleteByIds(response.deletedIds)
        // Save sync cursor
        prefs.saveSyncToken(response.syncToken)
    }
}
```

---

## CRDT-Inspired Patterns

```kotlin
// ✅ Grow-only Set (G-Set) — items can only be added, never removed
// Use for: tags, labels, read receipts

class GrowOnlySet<T>(private val items: MutableSet<T> = mutableSetOf()) {
    fun add(item: T) { items.add(item) }
    fun merge(other: GrowOnlySet<T>): GrowOnlySet<T> =
        GrowOnlySet((items + other.items).toMutableSet())
    fun contains(item: T) = items.contains(item)
}

// ✅ Last-Write-Wins Register (LWW-Register) — per-field versioning
data class LWWRegister<T>(val value: T, val timestamp: Long) {
    fun merge(other: LWWRegister<T>): LWWRegister<T> =
        if (other.timestamp > timestamp) other else this
}
```

---

## Anti-Patterns

- Replacing the entire local list with remote on every sync — loses local changes
- No tombstones for deletions — deleted items reappear after sync
- Using list position as merge key — positions shift, wrong items merged
- Merging without a base state — can't distinguish add from conflict
- Merging on the main thread — use Dispatchers.IO for large collections

---

## Related Skills

- `conflict-resolution` — single-record conflict handling
- `sync-engine` — orchestrating full sync with merge
- `room` — persisting merged results
- `dto-mapping` — mapping remote items for merge
