---
name: migration-strategy
description: >
  Migration strategy for Android apps — database, API, and data format migrations.
  Load this skill when planning Room database schema migrations, handling
  API version changes, migrating SharedPreferences to DataStore,
  or designing backward-compatible data format changes.
---

# Migration Strategy

## Overview
Migration strategy covers how the app evolves its data contracts without breaking existing users. This includes Room schema migrations (adding columns, renaming tables), API version transitions (v1 → v2 endpoints), and storage format migrations (SharedPreferences → DataStore, plain → encrypted). Each type requires a different approach.

---

## Core Principles

- **Never break existing users** — migrations must be backward-compatible or provide a clear upgrade path
- **Migrations are one-way** — don't design rollback unless specifically required
- **Test migrations** — always test the upgrade path from the previous version
- **Data loss is unacceptable** — destructive migration (`fallbackToDestructiveMigration`) is only for dev/cache data
- **Incremental** — migrate one version at a time; never skip versions

---

## Room Database Migration

```kotlin
// ✅ Version 1 → Version 2: Add column
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL(
            "ALTER TABLE users ADD COLUMN is_verified INTEGER NOT NULL DEFAULT 0"
        )
    }
}

// ✅ Version 2 → Version 3: Add new table
val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("""
            CREATE TABLE IF NOT EXISTS `user_sessions` (
                `id` TEXT NOT NULL,
                `user_id` TEXT NOT NULL,
                `token` TEXT NOT NULL,
                `expires_at` INTEGER NOT NULL,
                PRIMARY KEY(`id`),
                FOREIGN KEY(`user_id`) REFERENCES `users`(`id`) ON DELETE CASCADE
            )
        """.trimIndent())
        database.execSQL("CREATE INDEX IF NOT EXISTS `index_user_sessions_user_id` ON `user_sessions` (`user_id`)")
    }
}

// ✅ Version 3 → Version 4: Rename column (SQLite doesn't support RENAME COLUMN before API 29)
val MIGRATION_3_4 = object : Migration(3, 4) {
    override fun migrate(database: SupportSQLiteDatabase) {
        // SQLite < 3.25 workaround: create new table, copy, drop old
        database.execSQL("""
            CREATE TABLE `users_new` (
                `id` TEXT NOT NULL,
                `full_name` TEXT NOT NULL,
                `email` TEXT NOT NULL,
                `role` TEXT NOT NULL,
                `is_active` INTEGER NOT NULL,
                `is_verified` INTEGER NOT NULL DEFAULT 0,
                `created_at` INTEGER NOT NULL,
                `updated_at` INTEGER NOT NULL,
                PRIMARY KEY(`id`)
            )
        """.trimIndent())
        database.execSQL("""
            INSERT INTO `users_new`
            SELECT `id`, `name`, `email`, `role`, `is_active`, `is_verified`, `created_at`, `updated_at`
            FROM `users`
        """.trimIndent())
        database.execSQL("DROP TABLE `users`")
        database.execSQL("ALTER TABLE `users_new` RENAME TO `users`")
    }
}

// ✅ Register all migrations
@Database(entities = [...], version = 4)
abstract class AppDatabase : RoomDatabase() {
    companion object {
        fun build(context: Context): AppDatabase =
            Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
                .addMigrations(MIGRATION_1_2, MIGRATION_2_3, MIGRATION_3_4)
                .build()
    }
}
```

---

## Auto Migration (Room 2.4+)

```kotlin
// ✅ Simple migrations can use @AutoMigration
@Database(
    entities = [UserEntity::class],
    version = 5,
    autoMigrations = [
        AutoMigration(from = 4, to = 5)  // adds new columns with defaults automatically
    ]
)
abstract class AppDatabase : RoomDatabase()

// ✅ AutoMigration with spec for rename/delete
@RenameColumn(tableName = "users", fromColumnName = "name", toColumnName = "full_name")
class Migration4To5Spec : AutoMigrationSpec

@Database(
    version = 5,
    autoMigrations = [
        AutoMigration(from = 4, to = 5, spec = Migration4To5Spec::class)
    ]
)
abstract class AppDatabase : RoomDatabase()
```

---

## Testing Migrations

```kotlin
// ✅ Always test migration path
@RunWith(AndroidJUnit4::class)
class MigrationTest {

    @get:Rule
    val helper = MigrationTestHelper(
        InstrumentationRegistry.getInstrumentation(),
        AppDatabase::class.java
    )

    @Test
    fun migrate1To2() {
        // Create v1 database with test data
        helper.createDatabase("test.db", 1).apply {
            execSQL("INSERT INTO users VALUES ('1', 'Ali', 'ali@test.com', 'member', 1, 1000, 1000)")
            close()
        }

        // Run migration
        val db = helper.runMigrationsAndValidate("test.db", 2, true, MIGRATION_1_2)

        // Verify data intact + new column has default
        val cursor = db.query("SELECT * FROM users WHERE id = '1'")
        cursor.moveToFirst()
        assertThat(cursor.getInt(cursor.getColumnIndex("is_verified"))).isEqualTo(0)
        cursor.close()
    }
}
```

---

## SharedPreferences → DataStore Migration

```kotlin
// ✅ One-time migration from SharedPreferences to DataStore
class PreferencesMigration @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val legacyPrefs = context.getSharedPreferences("app_prefs", Context.MODE_PRIVATE)
    private val migrationKey = "prefs_migrated_v1"

    suspend fun migrateIfNeeded(dataStore: DataStore<Preferences>) {
        if (legacyPrefs.getBoolean(migrationKey, false)) return

        dataStore.edit { prefs ->
            // Copy all keys from SharedPreferences to DataStore
            legacyPrefs.getString("auth_token", null)?.let {
                prefs[stringPreferencesKey("auth_token")] = it
            }
            legacyPrefs.getBoolean("notifications_enabled", true).let {
                prefs[booleanPreferencesKey("notifications_enabled")] = it
            }
            legacyPrefs.getString("selected_language", "en")?.let {
                prefs[stringPreferencesKey("selected_language")] = it
            }
        }

        // Mark migration complete
        legacyPrefs.edit { putBoolean(migrationKey, true) }

        // Optionally clear old prefs
        legacyPrefs.edit { clear() }
    }
}

// ✅ Run at app startup — before DataStore is used
class App : Application() {
    @Inject lateinit var preferencesMigration: PreferencesMigration
    @Inject lateinit var dataStore: DataStore<Preferences>

    override fun onCreate() {
        super.onCreate()
        // Run migration in background
        applicationScope.launch {
            preferencesMigration.migrateIfNeeded(dataStore)
        }
    }
}
```

---

## API Version Migration

```kotlin
// ✅ Support multiple API versions via versioned endpoints
interface UserApiService {
    @GET("v2/users/{id}")
    suspend fun getUser(@Path("id") id: String): UserDtoV2

    // Keep v1 for gradual migration
    @GET("v1/users/{id}")
    suspend fun getUserLegacy(@Path("id") id: String): UserDtoV1
}

// ✅ Repository handles version negotiation
class UserRepositoryImpl @Inject constructor(
    private val api: UserApiService,
    private val featureFlags: FeatureFlags
) : UserRepository {

    override suspend fun getUser(id: String): Result<User> = runCatching {
        if (featureFlags.isApiV2Enabled) {
            api.getUser(id).toDomain()
        } else {
            api.getUserLegacy(id).toDomain()
        }
    }
}
```

---

## Data Format Migration

```kotlin
// ✅ Version-tagged serialized data
@Serializable
data class StoredData(
    val version: Int = CURRENT_VERSION,
    val payload: JsonElement
) {
    companion object {
        const val CURRENT_VERSION = 2

        fun migrate(stored: StoredData): StoredData = when (stored.version) {
            1 -> migrateV1ToV2(stored)
            CURRENT_VERSION -> stored
            else -> throw IllegalStateException("Unknown version: ${stored.version}")
        }

        private fun migrateV1ToV2(stored: StoredData): StoredData {
            val v1 = Json.decodeFromJsonElement<DataV1>(stored.payload)
            val v2 = DataV2(
                id = v1.id,
                fullName = "${v1.firstName} ${v1.lastName}",  // merge fields
                email = v1.email
            )
            return StoredData(version = 2, payload = Json.encodeToJsonElement(v2))
        }
    }
}
```

---

## Migration Checklist

```
Before releasing a migration:
  ✅ New @Entity version incremented in @Database
  ✅ Migration object written and tested
  ✅ Migration registered in Room.databaseBuilder
  ✅ MigrationTest written and passing
  ✅ Tested on device upgrading from previous version
  ✅ No data loss for existing users
  ✅ fallbackToDestructiveMigration NOT used (unless cache-only DB)
```

---

## Anti-Patterns

- `fallbackToDestructiveMigration()` for user data — wipes all data on missing migration
- Skipping migration versions — always chain migrations 1→2→3, never jump 1→3
- Not testing migration path — migration bugs only appear on upgrade, not fresh install
- Modifying old migrations — never change a released migration; add a new one
- Storing migration state in the migrated DB — if migration fails, the state check fails too

---

## Related Skills
- `room` — Room database setup
- `dao` — DAO patterns affected by migrations
- `entity-design` — entity changes that trigger migrations
- `datastore` — DataStore as migration target from SharedPreferences
- `gradle` — build versioning that tracks DB version
