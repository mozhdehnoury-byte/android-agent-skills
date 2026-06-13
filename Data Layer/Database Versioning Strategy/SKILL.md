---
name: database-versioning-strategy
description: >
  Strategy for managing Room database version increments across releases.
  Load this skill when planning schema changes, deciding when to increment
  version, managing version history, or coordinating schema changes across
  multiple developers/branches.
---

# Database Versioning Strategy

## Overview
Database versioning is the discipline of managing schema changes in a controlled, predictable way. Every schema change must be planned, versioned, documented, and tested before release. A broken migration path crashes the app on existing installs.

---

## Core Principles

- **One version increment per release** — batch all schema changes for a release into one migration
- **Never reuse a version number** — once a version is released, it's permanent
- **Never modify a released migration** — write a new one instead
- Export and commit the schema file — it's the source of truth for migration correctness
- Plan schema changes before implementation — migrations are hard to undo

---

## Version Numbering

```kotlin
// ✅ Increment by 1 for each release with schema changes
@Database(version = 5, ...)  // was 4 in previous release

// ✅ Never skip version numbers
// Bad: 1 → 3 (skipped 2) — users on v2 have no migration path to v3
// Good: 1 → 2 → 3

// ✅ During development — batch changes
// Don't increment version for every local change during a sprint
// Increment once when the feature/release is ready
```

---

## Schema Export and Version Control

```kotlin
// ✅ Always export schema
@Database(
    entities = [...],
    version = 5,
    exportSchema = true  // generates schemas/com.example.AppDatabase/5.json
)

// build.gradle.kts
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
}
```

```
// ✅ Commit schema files to version control
schemas/
  com.example.AppDatabase/
    1.json   ← version 1 schema
    2.json   ← version 2 schema
    3.json   ← version 3 schema
    4.json   ← version 4 schema
    5.json   ← version 5 schema (current)
```

---

## Change Planning Checklist

Before making any schema change:

```
☐ What tables/columns are affected?
☐ Is existing data preserved? How?
☐ Is the migration reversible? (Usually not — plan carefully)
☐ Are there indices that need updating?
☐ Are there related DAOs that need query updates?
☐ Are there DTOs/mappers that need updating?
☐ Is a data migration needed (not just schema migration)?
☐ Is this the only schema change for this release?
```

---

## Multi-Developer Coordination

```
// ✅ Strategy when multiple developers change schema simultaneously

// Developer A: adds column to users table
// Developer B: adds new orders table

// On merge:
// 1. Agree on a single version number for the combined change
// 2. Write ONE migration that includes BOTH changes
// 3. Never have two separate migrations with the same version increment

// ✅ Use a migration registry file to coordinate
// migrations/MIGRATION_LOG.md
// Version 5 (Release 2.3.0):
//   - Added phone_number to users (Developer A)
//   - Added orders table (Developer B)
```

---

## Data Migration vs Schema Migration

```kotlin
// Schema migration — changes structure
val MIGRATION_4_5 = object : Migration(4, 5) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN tier TEXT NOT NULL DEFAULT 'free'")
    }
}

// ✅ Data migration — backfill or transform data after schema change
// Do NOT do complex data transformations in Migration.migrate()
// Use a one-time WorkManager job instead

class BackfillUserTierWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        try {
            val users = userDao.getAllOnce()
            users.forEach { user ->
                val tier = computeTierFromHistory(user)
                userDao.updateTier(user.id, tier)
            }
            Result.success()
        } catch (e: Exception) {
            Result.retry()
        }
    }
}

// Schedule after migration is complete
fun scheduleBackfillIfNeeded(prefs: SharedPreferences) {
    if (!prefs.getBoolean("backfill_tier_done", false)) {
        WorkManager.getInstance(context).enqueueUniqueWork(
            "backfill_user_tier",
            ExistingWorkPolicy.KEEP,
            OneTimeWorkRequestBuilder<BackfillUserTierWorker>().build()
        )
    }
}
```

---

## Emergency: Breaking Schema in Development

```kotlin
// ✅ During development only — if migration is too complex and data doesn't matter
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .fallbackToDestructiveMigration()  // ❌ NEVER in production release
    .build()

// ✅ Better approach in development — clear app data manually
// adb shell pm clear com.example.app

// ✅ Or use a different DB name per build type
val dbName = if (BuildConfig.DEBUG) "app_database_debug" else "app_database"
```

---

## Version History Documentation

```markdown
<!-- schemas/CHANGELOG.md — maintain alongside schema files -->

## Version 5 (Release 2.3.0 — 2024-11-15)
- Added `phone_number TEXT` to `users` table
- Added `orders` table with foreign key to `users`
- Added index on `orders.user_id`

## Version 4 (Release 2.2.0 — 2024-09-01)
- Added `status TEXT NOT NULL DEFAULT 'active'` to `users`

## Version 3 (Release 2.0.0 — 2024-06-10)
- Renamed column `name` → `full_name` in `users`

## Version 2 (Release 1.5.0 — 2024-03-20)
- Added `created_at INTEGER` to `users`

## Version 1 (Initial release — 2024-01-10)
- Initial schema: `users` table
```

---

## Anti-Patterns

- Incrementing version during development for every small change — creates unnecessary migrations
- Not committing schema JSON files — migration tests fail, history is lost
- Writing migration without updating the version in `@Database` — causes crash
- Two migrations with overlapping version ranges — Room can't resolve path
- Complex business logic inside `Migration.migrate()` — use WorkManager instead
- Releasing without testing migration from the previous production version

---

## Related Skills
- `migration` — writing individual migration objects
- `room` — Room database configuration
- `workmanager` — data migration jobs post-schema-change
