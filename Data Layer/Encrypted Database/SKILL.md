---
name: encrypted-database
description: >
  Encrypted SQLite database using SQLCipher and Room for Android.
  Load this skill when storing sensitive user data that requires encryption
  at rest, implementing encrypted Room database, or managing encryption keys.
---

# Encrypted Database

## Overview

SQLCipher provides transparent AES-256 encryption for SQLite databases. When integrated with Room, the entire database file is encrypted at rest. This is required for apps handling sensitive data — health records, financial data, private messages, or enterprise security requirements.

---

## Core Principles

- Use SQLCipher when the database contains **sensitive user data**
- Encryption key must be stored in **Android Keystore** — never hardcoded
- Key derivation from user password uses PBKDF2 — never use raw password as key
- Encrypted database has a **performance overhead** — benchmark before requiring it
- Test encryption on real devices — emulators may behave differently

---

## Setup

```toml
[versions]
sqlcipher = "4.5.4"
security-crypto = "1.1.0-alpha06"

[libraries]
sqlcipher-android = { module = "net.zetetic:sqlcipher-android", version.ref = "sqlcipher" }
sqlite-ktx        = { module = "androidx.sqlite:sqlite-ktx", version = "2.4.0" }
security-crypto   = { module = "androidx.security:security-crypto", version.ref = "security-crypto" }
```

```kotlin
dependencies {
    implementation(libs.sqlcipher.android)
    implementation(libs.sqlite.ktx)
    implementation(libs.security.crypto)
}
```

---

## Key Management with Android Keystore

```kotlin
// ✅ Generate or retrieve encryption key from Keystore
object DatabaseKeyManager {

    private const val KEY_ALIAS = "db_encryption_key"
    private const val PREFS_NAME = "db_key_prefs"
    private const val PREFS_KEY = "encrypted_db_passphrase"

    fun getOrCreateKey(context: Context): ByteArray {
        val masterKey = MasterKey.Builder(context)
            .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
            .build()

        val prefs = EncryptedSharedPreferences.create(
            context,
            PREFS_NAME,
            masterKey,
            EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
            EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
        )

        val existing = prefs.getString(PREFS_KEY, null)
        if (existing != null) {
            return Base64.decode(existing, Base64.NO_WRAP)
        }

        // Generate new random key
        val newKey = ByteArray(32).also { SecureRandom().nextBytes(it) }
        prefs.edit()
            .putString(PREFS_KEY, Base64.encodeToString(newKey, Base64.NO_WRAP))
            .apply()
        return newKey
    }
}
```

---

## Room with SQLCipher

```kotlin
// ✅ Custom SupportSQLiteOpenHelper.Factory using SQLCipher
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        val passphrase = DatabaseKeyManager.getOrCreateKey(context)
        val factory = SupportFactory(passphrase)

        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "encrypted_app_database"
        )
        .openHelperFactory(factory)
        .addMigrations(MIGRATION_1_2)
        .build()
    }
}
```

---

## User-Password-Derived Key

```kotlin
// ✅ For apps where encryption key is derived from user's password
object PasswordKeyDerivation {

    fun deriveKey(password: CharArray, salt: ByteArray): ByteArray {
        val spec = PBEKeySpec(
            password,
            salt,
            100_000,  // iterations
            256       // key length in bits
        )
        val factory = SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256")
        return factory.generateSecret(spec).encoded
    }

    fun generateSalt(): ByteArray {
        return ByteArray(32).also { SecureRandom().nextBytes(it) }
    }
}

// Store the salt (not the key) in EncryptedSharedPreferences
// Re-derive the key each session from user's password
```

---

## Verifying Encryption

```kotlin
// ✅ Verify database is encrypted (for debug/test)
fun isDatabaseEncrypted(context: Context, dbName: String): Boolean {
    val dbFile = context.getDatabasePath(dbName)
    if (!dbFile.exists()) return false

    return try {
        val bytes = dbFile.readBytes().take(16).toByteArray()
        val header = String(bytes)
        !header.startsWith("SQLite format 3")  // encrypted files don't start with this
    } catch (e: Exception) {
        false
    }
}
```

---

## Migration with Encrypted Database

```kotlin
// ✅ Migrations work the same way — SQLCipher handles the encryption transparently
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN bio TEXT")
    }
}

// No special handling needed — Room + SQLCipher = transparent
```

---

## Performance Considerations

```kotlin
// ✅ SQLCipher overhead — benchmark before requiring encryption
// Typical overhead: 5-15% slower reads, 5-20% slower writes
// Acceptable for most use cases — unacceptable for high-frequency sensors/logs

// ✅ If encryption overhead is a concern:
// - Encrypt only the sensitive tables using separate database files
// - Use per-row encryption for specific columns instead of full-DB encryption
// - Profile with Macrobenchmark before deciding
```

---

## Anti-Patterns

- Hardcoding the encryption passphrase in code — visible in decompiled APK
- Using the user's password directly as the DB key — weak, changes break DB
- Not storing the salt — can't re-derive key from password
- Not using Android Keystore to protect the DB key — key accessible on rooted devices
- Encrypting non-sensitive data — unnecessary performance overhead
- Sharing encrypted DB across users on the same device — key management nightmare

---

## Related Skills

- `room` — Room database setup
- `keystore` — Android Keystore for key storage
- `encrypted-storage` — EncryptedSharedPreferences for key storage
- `security` — general Android security patterns
