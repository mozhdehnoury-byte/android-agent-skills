---
name: migration
description: >
  Room database migration — writing, testing, and managing schema changes.
  Load this skill when changing the database schema (add/remove column or table,
  change type), writing Migration objects, or testing migrations.
---

# Migration

## Overview
A migration defines how to transform the database schema from one version to the next without losing user data. Room requires a migration path for every version increment. Missing migrations cause a crash at startup — Room detects the schema mismatch and throws an `IllegalStateException`.

---

## Core Principles

- **Never use `fallbackToDestructiveMigration()`** in production — it deletes all user data
- Always export schema (`exportSchema = true`) — required for writing correct migrations
- Write a migration for **every version increment** — even if only one table changed
- **Test every migration** — automated migration tests catch silent data corruption
- Keep migration SQL simple — complex logic belongs in a one-time data migration job

---

## Version Increment Workflow

```
1. Change @Entity — add/remove/rename column or table
2. Increment version in @Database
3. Write Migration(oldVersion, newVersion)
4. Add migration to RoomDatabase builder
5. Export schema and commit schemas/ directory
6. Write migration test
```

---

## Writing Migrations

```kotlin
// ✅ Add a column
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE users ADD COLUMN phone_number TEXT"
        )
    }
}

// ✅ Add a column with default value
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE users ADD COLUMN status TEXT NOT NULL DEFAULT 'active'"
        )
    }
}

// ✅ Add a new table
val MIGRATION_3_4 = object : Migration(3, 4) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("""
            CREATE TABLE IF NOT EXISTS orders (
                id TEXT NOT NULL PRIMARY KEY,
                user_id TEXT NOT NULL,
                total INTEGER NOT NULL,
                created_at INTEGER NOT NULL,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )
        """)
        database.execSQL(
            "CREATE INDEX IF NOT EXISTS index_orders_user_id ON orders(user_id)"
        )
    }
}

// ✅ Rename a column (SQLite doesn't support RENAME COLUMN before API 29)
val MIGRATION_4_5 = object : Migration(4, 5) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // Create new table with correct schema
        database.execSQL("""
            CREATE TABLE users_new (
                id TEXT NOT NULL PRIMARY KEY,
                full_name TEXT NOT NULL,
                email TEXT NOT NULL
            )
        """)
        // Copy data
        database.execSQL("""
            INSERT INTO users_new (id, full_name, email)
            SELECT id, name, email FROM users
        """)
        // Drop old table
        database.execSQL("DROP TABLE users")
        // Rename new table
        database.execSQL("ALTER TABLE users_new RENAME TO users")
    }
}

// ✅ Drop a column (same approach — recreate table)
val MIGRATION_5_6 = object : Migration(5, 6) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("""
            CREATE TABLE users_new (
                id TEXT NOT NULL PRIMARY KEY,
                full_name TEXT NOT NULL
                -- removed: phone_number
            )
        """)
        database.execSQL("INSERT INTO users_new SELECT id, full_name FROM users")
        database.execSQL("DROP TABLE users")
        database.execSQL("ALTER TABLE users_new RENAME TO users")
    }
}
```

---

## Registering Migrations

```kotlin
// ✅ Register all migrations in database builder
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
            .addMigrations(
                MIGRATION_1_2,
                MIGRATION_2_3,
                MIGRATION_3_4,
                MIGRATION_4_5,
                MIGRATION_5_6
            )
            .build()
}
```

---

## Auto Migration (Room 2.4+)

```kotlin
// ✅ For simple schema changes — Room generates migration automatically
@Database(
    entities = [UserEntity::class],
    version = 3,
    exportSchema = true,
    autoMigrations = [
        AutoMigration(from = 1, to = 2),  // simple add column — auto-generated
        AutoMigration(from = 2, to = 3, spec = Migration2To3::class)  // with rename
    ]
)
abstract class AppDatabase : RoomDatabase()

// ✅ AutoMigrationSpec — for rename/delete (needs disambiguation)
@RenameColumn(tableName = "users", fromColumnName = "name", toColumnName = "full_name")
class Migration2To3 : AutoMigrationSpec
```

---

## Testing Migrations

```kotlin
// ✅ MigrationTestHelper — test each migration
@RunWith(AndroidJUnit4::class)
class MigrationTest {

    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java
    )

    @Test
    fun migrate1To2() {
        // Create version 1 database
        helper.createDatabase(TEST_DB, 1).apply {
            execSQL("INSERT INTO users VALUES ('1', 'Ali')")
            close()
        }

        // Run migration
        val db = helper.runMigrationsAndValidate(TEST_DB, 2, true, MIGRATION_1_2)

        // Verify data integrity
        val cursor = db.query("SELECT * FROM users WHERE id = '1'")
        assertTrue(cursor.moveToFirst())
        assertEquals("Ali", cursor.getString(cursor.getColumnIndex("full_name")))
        // new column exists with default value
        assertNotNull(cursor.getString(cursor.getColumnIndex("phone_number")))
        cursor.close()
    }

    companion object {
        private const val TEST_DB = "migration_test"
    }
}
```

---

## Multi-Step Migration Path

```kotlin
// ✅ Room applies migrations sequentially — 1→2→3→4 not 1→4
// All intermediate migrations must be present

// If user has version 1, Room applies:
// MIGRATION_1_2, then MIGRATION_2_3, then MIGRATION_3_4

// ✅ Multi-version single migration (skip versions)
val MIGRATION_1_4 = object : Migration(1, 4) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // Apply all changes from 1→4 in one step
        // Only needed if you want to optimize multi-version upgrades
    }
}
```

---

## Anti-Patterns

- `fallbackToDestructiveMigration()` in production — deletes all user data
- Not exporting schema (`exportSchema = false`) — can't write correct migrations
- Running data transformations in migration SQL — use a one-time WorkManager job instead
- Not committing `schemas/` directory to version control — migration tests require it
- Incrementing version without writing migration — crashes on existing installs
- Writing migration without a test — silent data corruption risk

---

## Related Skills
- `room` — Room database setup
- `dao` — DAO queries after schema changes
- `database-versioning-strategy` — when and how to version the schema
- `workmanager` — one-time data migration jobs after schema change
