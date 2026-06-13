---
name: sqlite
description: >
  SQLite fundamentals and direct SQLite usage in Android.
  Load this skill when working with raw SQLite queries, SupportSQLiteDatabase
  in migrations, understanding SQLite constraints, or diagnosing SQL issues
  that go beyond Room's abstraction.
---

# SQLite

## Overview
SQLite is the embedded relational database engine used by Android. Room is built on top of SQLite and handles most use cases. Direct SQLite access is needed in migrations, performance diagnostics, and edge cases that Room doesn't cover. Understanding SQLite's constraints prevents data integrity bugs.

---

## Core Principles

- Prefer Room over raw SQLite — use raw SQLite only when Room is insufficient
- SQLite is **not thread-safe** — always use Room's dispatcher management
- SQLite uses **dynamic typing** — enforce types via constraints, not the engine
- SQLite does **not enforce foreign keys by default** — must be enabled explicitly
- Transactions are the primary tool for performance and atomicity

---

## SQLite Type System

```sql
-- SQLite storage classes (not strict types)
NULL     -- null value
INTEGER  -- signed integer (1, 2, 3, 4, 6, or 8 bytes)
REAL     -- floating point (8-byte IEEE 754)
TEXT     -- UTF-8/16 string
BLOB     -- binary data

-- Room maps Kotlin types to SQLite:
-- String  → TEXT
-- Int     → INTEGER
-- Long    → INTEGER
-- Double  → REAL
-- Boolean → INTEGER (0 or 1)
-- ByteArray → BLOB
```

---

## Foreign Keys — Must Be Enabled

```kotlin
// ✅ Enable foreign key enforcement in Room
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addCallback(object : RoomDatabase.Callback() {
        override fun onOpen(db: SupportSQLiteDatabase) {
            db.execSQL("PRAGMA foreign_keys = ON")
        }
    })
    .build()

// Without this, foreign key constraints are silently ignored
```

---

## Direct SQL in Migrations

```kotlin
val MIGRATION_3_4 = object : Migration(3, 4) {
    override fun migrate(database: SupportSQLiteDatabase) {

        // ✅ Create table
        database.execSQL("""
            CREATE TABLE IF NOT EXISTS products (
                id TEXT NOT NULL PRIMARY KEY,
                name TEXT NOT NULL,
                price INTEGER NOT NULL DEFAULT 0,
                category TEXT NOT NULL,
                created_at INTEGER NOT NULL
            )
        """)

        // ✅ Add column
        database.execSQL("ALTER TABLE users ADD COLUMN avatar_url TEXT")

        // ✅ Create index
        database.execSQL(
            "CREATE INDEX IF NOT EXISTS index_products_category ON products(category)"
        )

        // ✅ Create unique index
        database.execSQL(
            "CREATE UNIQUE INDEX IF NOT EXISTS index_users_email ON users(email)"
        )

        // ✅ Drop index
        database.execSQL("DROP INDEX IF EXISTS old_index_name")

        // ✅ Drop table
        database.execSQL("DROP TABLE IF EXISTS old_table")

        // ✅ Rename table (within migration)
        database.execSQL("ALTER TABLE users_temp RENAME TO users")
    }
}
```

---

## Rename Column (Pre-API 29)

```kotlin
// SQLite supports RENAME COLUMN only from API 29 / SQLite 3.25.0
// For older support — recreate table

val MIGRATION_RENAME = object : Migration(5, 6) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // 1. Create new table with correct schema
        database.execSQL("""
            CREATE TABLE users_new (
                id TEXT NOT NULL PRIMARY KEY,
                full_name TEXT NOT NULL,   -- renamed from 'name'
                email TEXT NOT NULL,
                created_at INTEGER NOT NULL
            )
        """)
        // 2. Copy data
        database.execSQL("""
            INSERT INTO users_new (id, full_name, email, created_at)
            SELECT id, name, email, created_at FROM users
        """)
        // 3. Drop old table
        database.execSQL("DROP TABLE users")
        // 4. Rename new table
        database.execSQL("ALTER TABLE users_new RENAME TO users")
        // 5. Recreate indices
        database.execSQL("CREATE INDEX index_users_email ON users(email)")
    }
}
```

---

## Transactions for Performance

```kotlin
// ✅ Wrap bulk inserts in a transaction — dramatically faster
val MIGRATION_BULK = object : Migration(6, 7) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.beginTransaction()
        try {
            database.execSQL("ALTER TABLE orders ADD COLUMN tax INTEGER NOT NULL DEFAULT 0")
            // backfill
            database.execSQL("UPDATE orders SET tax = CAST(total * 0.09 AS INTEGER)")
            database.setTransactionSuccessful()
        } finally {
            database.endTransaction()
        }
    }
}

// ✅ Room handles transactions via @Transaction in DAOs
// Use raw transactions only in migrations or SupportSQLiteDatabase contexts
```

---

## Useful PRAGMA Commands

```sql
-- Check schema integrity
PRAGMA integrity_check;

-- Get table info (columns, types, constraints)
PRAGMA table_info(users);

-- List all tables
SELECT name FROM sqlite_master WHERE type='table';

-- List all indices
SELECT name FROM sqlite_master WHERE type='index';

-- Get foreign key list for a table
PRAGMA foreign_key_list(orders);

-- Check foreign key violations
PRAGMA foreign_key_check;

-- Get current database version (Room version)
PRAGMA user_version;

-- WAL mode — better concurrent read performance
PRAGMA journal_mode = WAL;
```

---

## WAL Mode

```kotlin
// ✅ Enable WAL (Write-Ahead Logging) for better read concurrency
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .setJournalMode(RoomDatabase.JournalMode.WRITE_AHEAD_LOGGING)
    .build()

// WAL benefits:
// - Readers don't block writers
// - Writers don't block readers
// - Better performance for concurrent access patterns
// Default in Room — explicitly set if needed
```

---

## Querying System Tables

```kotlin
// ✅ Check if a table exists (useful in migrations)
fun SupportSQLiteDatabase.tableExists(tableName: String): Boolean {
    val cursor = query(
        "SELECT name FROM sqlite_master WHERE type='table' AND name=?",
        arrayOf(tableName)
    )
    return cursor.use { it.count > 0 }
}

// ✅ Check if a column exists (useful in migrations)
fun SupportSQLiteDatabase.columnExists(tableName: String, columnName: String): Boolean {
    val cursor = query("PRAGMA table_info($tableName)", null)
    return cursor.use { c ->
        while (c.moveToNext()) {
            if (c.getString(c.getColumnIndex("name")) == columnName) return true
        }
        false
    }
}
```

---

## Anti-Patterns

- Raw SQLite without Room — loses type safety, compile-time checks, and Flow support
- Not enabling foreign keys (`PRAGMA foreign_keys = ON`) — constraints silently ignored
- String concatenation in SQL — use parameterized queries to prevent SQL injection
- Bulk operations outside transactions — orders of magnitude slower
- Using `AUTOINCREMENT` keyword — use `INTEGER PRIMARY KEY` instead (simpler, faster)
- Assuming column order is stable — always use column names, not positions

---

## Related Skills
- `room` — Room as the primary SQLite abstraction
- `dao` — Room DAO for type-safe queries
- `migration` — using SupportSQLiteDatabase in migrations
- `indexing` — index creation and management
