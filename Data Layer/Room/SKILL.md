---
name: room
description: >
  Room database setup, configuration, and best practices for Android.
  Load this skill when setting up Room, defining entities, configuring
  the database class, handling migrations, or integrating Room with
  repositories and coroutines/Flow.
---

# Room

## Overview
Room is Android's SQLite abstraction library. It provides compile-time SQL verification, coroutine/Flow support, and migration tooling. Room is the standard local database solution for Android and works as the offline-first data source in a Repository pattern.

---

## Core Principles

- Room operations must always run on **Dispatchers.IO** — never main thread
- Expose **Flow** from DAOs for observable queries — never suspend for reactive data
- Use **suspend** for one-shot write operations (insert, update, delete)
- Never expose entities outside the data layer — map to domain models in Repository
- Always define **indices** on columns used in WHERE and JOIN clauses

---

## Setup

```toml
# libs.versions.toml
[versions]
room = "2.6.1"

[libraries]
room-runtime = { module = "androidx.room:room-runtime", version.ref = "room" }
room-ktx     = { module = "androidx.room:room-ktx", version.ref = "room" }
room-compiler = { module = "androidx.room:room-compiler", version.ref = "room" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version = "2.0.0-1.0.21" }
```

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.ksp)
}

dependencies {
    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)
}

// ✅ Schema export — for migration verification
ksp {
    arg("room.schemaLocation", "$projectDir/schemas")
    arg("room.incremental", "true")
}
```

---

## Entity

```kotlin
// ✅ Entity — maps to a database table
@Entity(
    tableName = "users",
    indices = [
        Index(value = ["email"], unique = true),
        Index(value = ["status"])
    ]
)
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name") val name: String,
    @ColumnInfo(name = "email") val email: String,
    @ColumnInfo(name = "status") val status: String,
    @ColumnInfo(name = "created_at") val createdAt: Long,
    @ColumnInfo(name = "updated_at") val updatedAt: Long
)

// ✅ Embedded object
data class Address(
    val street: String,
    val city: String,
    val country: String
)

@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    val name: String,
    @Embedded val address: Address
)

// ✅ Relation — one-to-many
data class UserWithOrders(
    @Embedded val user: UserEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val orders: List<OrderEntity>
)
```

---

## Database Class

```kotlin
// ✅ Database definition
@Database(
    entities = [
        UserEntity::class,
        OrderEntity::class,
        ProductEntity::class
    ],
    version = 3,
    exportSchema = true  // always true — needed for migration testing
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {

    abstract fun userDao(): UserDao
    abstract fun orderDao(): OrderDao
    abstract fun productDao(): ProductDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                .fallbackToDestructiveMigrationOnDowngrade()
                .build()
                .also { INSTANCE = it }
            }
        }
    }
}

// ✅ With Hilt — preferred over manual singleton
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
            .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
            .build()

    @Provides
    fun provideUserDao(database: AppDatabase): UserDao = database.userDao()
}
```

---

## Type Converters

```kotlin
// ✅ For types Room can't store natively
class Converters {

    @TypeConverter
    fun fromTimestamp(value: Long?): Date? =
        value?.let { Date(it) }

    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? =
        date?.time

    @TypeConverter
    fun fromStringList(value: String): List<String> =
        Json.decodeFromString(value)

    @TypeConverter
    fun toStringList(list: List<String>): String =
        Json.encodeToString(list)
}
```

---

## Integration with Repository

```kotlin
// ✅ Repository maps Entity ↔ Domain model
class UserRepository @Inject constructor(
    private val userDao: UserDao,
    private val userMapper: UserMapper
) {

    // Observable query — Flow
    fun observeUsers(): Flow<List<User>> =
        userDao.observeAll()
            .map { entities -> entities.map { userMapper.toDomain(it) } }

    // One-shot read
    suspend fun getUserById(id: String): User? =
        userDao.getById(id)?.let { userMapper.toDomain(it) }

    // Write operations
    suspend fun saveUser(user: User) {
        userDao.upsert(userMapper.toEntity(user))
    }

    suspend fun deleteUser(id: String) {
        userDao.deleteById(id)
    }
}
```

---

## Anti-Patterns

- Running Room queries on the main thread — causes ANR
- Exposing `UserEntity` outside the data layer — couples UI to DB schema
- Using `allowMainThreadQueries()` — only acceptable in tests
- Not exporting schema (`exportSchema = false`) — can't write or test migrations
- Missing indices on queried columns — slow queries on large datasets
- Using `@PrimaryKey(autoGenerate = true)` with String IDs — use UUID instead
- Accessing database instance directly from ViewModel — always go through Repository

---

## Related Skills
- `dao` — DAO queries and operations
- `migration` — database schema migrations
- `dto-mapping` — Entity ↔ Domain mapping
- `repository-pattern` — Repository wrapping Room
- `hilt` — injecting Room database and DAOs
