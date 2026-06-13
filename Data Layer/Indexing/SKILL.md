---
name: indexing
description: >
  SQLite/Room database indexing strategy for Android.
  Load this skill when defining indices on entities, diagnosing slow queries,
  or deciding which columns need indexing for query performance.
---

# Indexing

## Overview

A database index is a data structure that speeds up query lookups at the cost of extra storage and slower writes. Without proper indices, queries perform a full table scan — acceptable for small tables, catastrophic for large ones. In Room, indices are declared on `@Entity`.

---

## Core Principles

- Index columns used in **WHERE**, **JOIN ON**, and **ORDER BY** clauses
- Index foreign key columns — Room/SQLite does not do this automatically
- **Don't over-index** — every index slows down INSERT/UPDATE/DELETE
- Unique indices enforce data integrity at the DB level
- Composite indices cover multi-column WHERE clauses

---

## Declaring Indices in Room

```kotlin
// ✅ Single column index
@Entity(
    tableName = "users",
    indices = [
        Index(value = ["email"], unique = true),  // unique constraint
        Index(value = ["status"]),                 // for filtering
        Index(value = ["created_at"])              // for sorting
    ]
)
data class UserEntity(
    @PrimaryKey val id: String,
    val email: String,
    val status: String,
    @ColumnInfo(name = "created_at") val createdAt: Long
)

// ✅ Composite index — covers multi-column queries
@Entity(
    tableName = "orders",
    indices = [
        Index(value = ["user_id", "status"]),  // WHERE user_id = ? AND status = ?
        Index(value = ["user_id"])              // WHERE user_id = ? alone also uses this
    ]
)
data class OrderEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "user_id") val userId: String,
    val status: String,
    val total: Int
)
```

---

## When to Add an Index

```kotlin
// ✅ Index needed — WHERE on non-primary-key column
@Query("SELECT * FROM users WHERE status = :status")
// → index on status

// ✅ Index needed — ORDER BY on non-primary-key column
@Query("SELECT * FROM users ORDER BY created_at DESC")
// → index on created_at

// ✅ Index needed — JOIN condition
@Query("SELECT * FROM orders o JOIN users u ON o.user_id = u.id")
// → index on orders.user_id (foreign key)

// ✅ Index needed — LIKE prefix search
@Query("SELECT * FROM users WHERE full_name LIKE :prefix || '%'")
// → index on full_name (helps prefix search, not suffix)

// ❌ Index NOT needed — primary key is already indexed
@Query("SELECT * FROM users WHERE id = :id")
// → no extra index needed
```

---

## Index Decision Matrix

| Query Pattern                  | Index Type                     |
| ------------------------------ | ------------------------------ |
| `WHERE col = ?`                | Single index on `col`          |
| `WHERE col1 = ? AND col2 = ?`  | Composite index `(col1, col2)` |
| `ORDER BY col`                 | Single index on `col`          |
| `WHERE col1 = ? ORDER BY col2` | Composite index `(col1, col2)` |
| `LIKE 'prefix%'`               | Single index on `col`          |
| Foreign key column             | Single index (always)          |
| Unique constraint              | Unique index                   |

---

## Foreign Key Indices

```kotlin
// ✅ Always index foreign key columns — SQLite doesn't auto-index them
@Entity(
    tableName = "orders",
    foreignKeys = [
        ForeignKey(
            entity = UserEntity::class,
            parentColumns = ["id"],
            childColumns = ["user_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [
        Index(value = ["user_id"])  // ✅ required — without this, cascades are slow
    ]
)
data class OrderEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "user_id") val userId: String
)
```

---

## Diagnosing Missing Indices

```kotlin
// ✅ Enable Room query logging in debug
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .setQueryCallback(
        queryCallback = { sqlQuery, bindArgs ->
            Log.d("RoomQuery", "Query: $sqlQuery Args: $bindArgs")
        },
        executor = Executors.newSingleThreadExecutor()
    )
    .build()

// ✅ Use EXPLAIN QUERY PLAN to check index usage
// Run in a test or via ADB:
// EXPLAIN QUERY PLAN SELECT * FROM users WHERE status = 'active'
// Look for "SCAN TABLE" (no index) vs "SEARCH TABLE USING INDEX" (indexed)
```

---

## Composite Index Column Order

```kotlin
// ✅ Put the most selective / most frequently filtered column FIRST
Index(value = ["user_id", "status"])
// Best for: WHERE user_id = ? AND status = ?
// Also good for: WHERE user_id = ?
// NOT useful for: WHERE status = ? alone

// ✅ For range queries, range column goes LAST
Index(value = ["user_id", "created_at"])
// Good for: WHERE user_id = ? AND created_at > ?
// Not for: WHERE created_at > ? alone
```

---

## Anti-Patterns

- No index on foreign key columns — DELETE/CASCADE operations become full table scans
- Index on every column — slows writes, wastes storage, rarely helps queries
- Composite index with wrong column order — first column must be used in WHERE
- Index on low-cardinality columns alone (e.g., boolean) — rarely helps (only 2 values)
- Missing index on ORDER BY column in large tables — full sort every query
- Unique index as a substitute for proper validation — validate in app layer too

---

## Related Skills

- `room` — Room entity and database setup
- `dao` — queries that benefit from indices
- `sqlite` — raw SQLite index management
- `database-versioning-strategy` — adding indices via migration
