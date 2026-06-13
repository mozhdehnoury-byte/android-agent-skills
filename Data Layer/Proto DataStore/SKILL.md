---
name: proto-datastore
description: >
  Proto DataStore setup and usage for typed object persistence in Android.
  Load this skill when storing complex structured data with type safety,
  using Protobuf schemas, or when Preferences DataStore is insufficient.
---

# Proto DataStore

## Overview
Proto DataStore stores typed objects defined by Protobuf schemas. Unlike Preferences DataStore (which uses string keys), Proto DataStore is strongly typed, schema-enforced, and forward-compatible by design. It is ideal for storing structured settings or state that evolves over time.

---

## Core Principles

- Use Proto DataStore when **type safety and schema evolution** matter
- Protobuf schemas define the contract — never change field numbers once released
- Like Preferences DataStore — **one instance per file**, inject via Hilt
- Always handle **IOException** in Flow collection
- Add new fields to the proto — never remove or renumber existing fields

---

## Setup

```toml
# libs.versions.toml
[versions]
datastore = "1.1.1"
protobuf = "4.26.1"
protobuf-plugin = "0.9.4"

[libraries]
datastore-proto = { module = "androidx.datastore:datastore", version.ref = "datastore" }
protobuf-javalite = { module = "com.google.protobuf:protobuf-javalite", version.ref = "protobuf" }
protobuf-kotlin-lite = { module = "com.google.protobuf:protobuf-kotlin-lite", version.ref = "protobuf" }

[plugins]
protobuf = { id = "com.google.protobuf", version.ref = "protobuf-plugin" }
```

```kotlin
// build.gradle.kts
plugins {
    alias(libs.plugins.protobuf)
}

dependencies {
    implementation(libs.datastore.proto)
    implementation(libs.protobuf.javalite)
    implementation(libs.protobuf.kotlin.lite)
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:${libs.versions.protobuf.get()}"
    }
    generateProtoTasks {
        all().forEach { task ->
            task.builtins {
                create("java") { option("lite") }
                create("kotlin") { option("lite") }
            }
        }
    }
}
```

---

## Proto Schema Definition

```protobuf
// src/main/proto/user_preferences.proto
syntax = "proto3";
option java_package = "com.example.app.datastore";
option java_multiple_files = true;

message UserPreferences {
    bool is_dark_mode = 1;
    string language = 2;
    int32 font_size = 3;
    SortOrder sort_order = 4;
    int64 last_sync_timestamp = 5;
    // ✅ New fields added here — never remove or renumber existing
}

enum SortOrder {
    SORT_ORDER_UNSPECIFIED = 0;
    SORT_ORDER_ASC = 1;
    SORT_ORDER_DESC = 2;
}
```

---

## Serializer

```kotlin
// ✅ Implement Serializer for the proto type
object UserPreferencesSerializer : Serializer<UserPreferences> {

    override val defaultValue: UserPreferences = UserPreferences.getDefaultInstance()

    override suspend fun readFrom(input: InputStream): UserPreferences {
        try {
            return UserPreferences.parseFrom(input)
        } catch (exception: InvalidProtocolBufferException) {
            throw CorruptionException("Cannot read proto.", exception)
        }
    }

    override suspend fun writeTo(t: UserPreferences, output: OutputStream) {
        t.writeTo(output)
    }
}
```

---

## Creating DataStore

```kotlin
// ✅ Create via extension property
val Context.userPreferencesDataStore: DataStore<UserPreferences> by dataStore(
    fileName = "user_preferences.pb",
    serializer = UserPreferencesSerializer
)

// ✅ Provide via Hilt
@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {

    @Provides
    @Singleton
    fun provideUserPreferencesDataStore(
        @ApplicationContext context: Context
    ): DataStore<UserPreferences> = context.userPreferencesDataStore
}
```

---

## Repository

```kotlin
class UserPreferencesRepository @Inject constructor(
    private val dataStore: DataStore<UserPreferences>
) {

    // ✅ Read — expose as Flow
    val userPreferences: Flow<UserPreferences> = dataStore.data
        .catch { e ->
            if (e is IOException) emit(UserPreferences.getDefaultInstance())
            else throw e
        }

    val isDarkMode: Flow<Boolean> = userPreferences.map { it.isDarkMode }
    val language: Flow<String> = userPreferences.map { it.language }

    // ✅ Write — use updateData or update (proto-specific)
    suspend fun setDarkMode(enabled: Boolean) {
        dataStore.updateData { current ->
            current.toBuilder()
                .setIsDarkMode(enabled)
                .build()
        }
    }

    suspend fun setLanguage(language: String) {
        dataStore.updateData { current ->
            current.toBuilder()
                .setLanguage(language)
                .build()
        }
    }

    // ✅ Kotlin DSL style (with protobuf-kotlin-lite)
    suspend fun updatePreferences(update: UserPreferences.Builder.() -> Unit) {
        dataStore.updateData { current ->
            current.copy(update)
        }
    }

    // Usage:
    // preferencesRepository.updatePreferences {
    //     isDarkMode = true
    //     language = "fa"
    // }
}
```

---

## Schema Evolution Rules

```protobuf
// ✅ Safe changes
// - Add new optional fields with new field numbers
// - Add new enum values (with default 0 for unknown)

// ❌ Breaking changes — NEVER do these to a released schema
// - Remove a field
// - Rename a field (the name is cosmetic in proto3, but confusing)
// - Change a field's type
// - Reuse a field number for a different field

// ✅ Example: safe evolution
message UserPreferences {
    bool is_dark_mode = 1;
    string language = 2;
    int32 font_size = 3;       // existing
    SortOrder sort_order = 4;  // existing
    // New field in v2 — safe to add
    bool notifications_enabled = 5;
    string theme_color = 6;
}
```

---

## Proto DataStore vs Preferences DataStore

| | Proto DataStore | Preferences DataStore |
|--|----------------|----------------------|
| Type safety | ✅ Strongly typed | ⚠️ Key-based |
| Schema evolution | ✅ Protobuf rules | ❌ Manual |
| Setup complexity | ⚠️ Higher | ✅ Simple |
| Best for | Structured objects | Simple settings |

---

## Anti-Patterns

- Reusing or renumbering proto field numbers — corrupts data for existing users
- Removing fields from proto — breaks deserialization of existing data
- Not providing a default value in Serializer — crashes on empty/new install
- Multiple DataStore instances for the same file — data corruption
- Storing large binary data in proto — use files or Room instead

---

## Related Skills
- `datastore` — Preferences DataStore for simple key-value storage
- `hilt` — singleton DataStore provision
- `repository-pattern` — wrapping DataStore in a Repository
- `key-value-store-strategy` — choosing between DataStore variants
