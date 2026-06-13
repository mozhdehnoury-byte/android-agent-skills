---
name: dao
description: >
  Room DAO design — queries, inserts, updates, deletes, transactions, and Flow.
  Load this skill when writing DAO interfaces, designing SQL queries,
  handling reactive queries with Flow, or performing batch operations.
---

# DAO (Data Access Object)

## Overview
A DAO is the interface through which the app accesses the Room database. It defines the contract between the app and the database — what queries to run, what data to return, and how writes happen. DAOs are generated at compile time and SQL is verified at compile time.

---

## Core Principles

- DAOs are **interfaces or abstract classes** — never concrete classes
- Use **Flow** for observable queries — use **suspend** for one-shot operations
- Prefer **upsert** over separate insert + update logic
- Keep queries **focused** — one query, one purpose
- Use **@Transaction** for operations that must be atomic

---

## Basic DAO Structure

```kotlin
@Dao
interface UserDao {

    // ✅ Observable query — Flow
    @Query("SELECT * FROM users ORDER BY full_name ASC")
    fun observeAll(): Flow<List<UserEntity>>

    // ✅ Observable single item
    @Query("SELECT * FROM users WHERE id = :id")
    fun observeById(id: String): Flow<UserEntity?>

    // ✅ One-shot query
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun getById(id: String): UserEntity?

    // ✅ Upsert — insert or replace on conflict
    @Upsert
    suspend fun upsert(user: UserEntity)

    @Upsert
    suspend fun upsertAll(users: List<UserEntity>)

    // ✅ Insert with conflict strategy
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertIfNotExists(user: UserEntity)

    // ✅ Update
    @Update
    suspend fun update(user: UserEntity)

    // ✅ Delete by entity
    @Delete
    suspend fun delete(user: UserEntity)

    // ✅ Delete by ID
    @Query("DELETE FROM users WHERE id = :id")
    suspend fun deleteById(id: String)

    // ✅ Delete all
    @Query("DELETE FROM users")
    suspend fun deleteAll()
}
```

---

## Filtered and Sorted Queries

```kotlin
@Dao
interface UserDao {

    // ✅ Filter by column
    @Query("SELECT * FROM users WHERE status = :status ORDER BY full_name ASC")
    fun observeByStatus(status: String): Flow<List<UserEntity>>

    // ✅ Search with LIKE
    @Query("SELECT * FROM users WHERE full_name LIKE '%' || :query || '%' OR email LIKE '%' || :query || '%'")
    fun search(query: String): Flow<List<UserEntity>>

    // ✅ Filter with IN clause
    @Query("SELECT * FROM users WHERE id IN (:ids)")
    suspend fun getByIds(ids: List<String>): List<UserEntity>

    // ✅ Count
    @Query("SELECT COUNT(*) FROM users WHERE status = :status")
    fun observeCountByStatus(status: String): Flow<Int>

    // ✅ Exists check
    @Query("SELECT EXISTS(SELECT 1 FROM users WHERE email = :email)")
    suspend fun existsByEmail(email: String): Boolean

    // ✅ Pagination with LIMIT and OFFSET
    @Query("SELECT * FROM users ORDER BY created_at DESC LIMIT :limit OFFSET :offset")
    suspend fun getPage(limit: Int, offset: Int): List<UserEntity>
}
```

---

## Joins and Relations

```kotlin
@Dao
interface UserDao {

    // ✅ Join query — returns a POJO (not entity)
    @Query("""
        SELECT u.*, o.id as order_id, o.total as order_total
        FROM users u
        INNER JOIN orders o ON u.id = o.user_id
        WHERE u.id = :userId
    """)
    suspend fun getUserWithOrders(userId: String): UserWithOrdersEntity?

    // ✅ @Transaction for relation queries
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithProfile(userId: String): UserWithProfileEntity?

    // ✅ @Transaction for all @Relation queries
    @Transaction
    @Query("SELECT * FROM users")
    fun observeAllWithOrders(): Flow<List<UserWithOrdersEntity>>
}
```

---

## Transactions

```kotlin
@Dao
interface UserDao {

    // ✅ @Transaction — atomic multi-step operation
    @Transaction
    suspend fun replaceAll(users: List<UserEntity>) {
        deleteAll()
        upsertAll(users)
    }

    @Transaction
    suspend fun transferOrders(fromUserId: String, toUserId: String) {
        updateOrderOwner(fromUserId, toUserId)
        updateUserOrderCount(fromUserId)
        updateUserOrderCount(toUserId)
    }

    @Query("UPDATE orders SET user_id = :toUserId WHERE user_id = :fromUserId")
    suspend fun updateOrderOwner(fromUserId: String, toUserId: String)

    @Query("UPDATE users SET order_count = (SELECT COUNT(*) FROM orders WHERE user_id = :userId) WHERE id = :userId")
    suspend fun updateUserOrderCount(userId: String)
}
```

---

## Return Types Reference

| Operation | Return Type |
|-----------|------------|
| Observable query (list) | `Flow<List<Entity>>` |
| Observable query (single) | `Flow<Entity?>` |
| Observable count | `Flow<Int>` |
| One-shot query | `suspend fun`: `Entity?`, `List<Entity>` |
| Insert | `suspend fun`: `Long` (row id) or `Unit` |
| Upsert | `suspend fun`: `Unit` |
| Update | `suspend fun`: `Int` (rows affected) or `Unit` |
| Delete | `suspend fun`: `Int` (rows affected) or `Unit` |

---

## POJOs for Custom Queries

```kotlin
// ✅ POJO for projection queries — not an @Entity
data class UserSummary(
    val id: String,
    @ColumnInfo(name = "full_name") val name: String,
    @ColumnInfo(name = "order_count") val orderCount: Int
)

@Dao
interface UserDao {
    @Query("""
        SELECT u.id, u.full_name, COUNT(o.id) as order_count
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
        GROUP BY u.id
        ORDER BY order_count DESC
    """)
    fun observeUserSummaries(): Flow<List<UserSummary>>
}
```

---

## Anti-Patterns

- Returning entities directly from Repository — always map to domain models
- Using `LiveData` instead of `Flow` in new code — Flow is preferred
- Missing `@Transaction` on `@Relation` queries — causes inconsistent data reads
- Using `suspend` for observable queries — use `Flow` instead
- Writing business logic in SQL queries — keep queries data-focused
- Not using `@Upsert` when insert-or-update is needed — avoids conflict handling mistakes
- Large `IN` clauses with dynamic lists — SQLite has limits; batch if needed

---

## Related Skills
- `room` — Room database setup and configuration
- `migration` — schema changes and migrations
- `dto-mapping` — Entity ↔ Domain model mapping
- `repository-pattern` — Repository wrapping DAOs
- `coroutine` — Dispatchers.IO for DAO operations
