---
name: encrypted-storage
description: >
  Encrypted local storage for Android apps.
  Load this skill when storing sensitive data (tokens, user credentials,
  personal info) locally, using EncryptedSharedPreferences, EncryptedFile,
  or integrating encryption with DataStore or Room.
---

# Encrypted Storage

## Overview
Encrypted Storage protects sensitive data at rest on the device. Android provides `EncryptedSharedPreferences` and `EncryptedFile` via the Security library, both backed by the Android Keystore. For DataStore and Room, encryption is applied at the file or database level.

---

## Core Principles

- **Never store sensitive data in plain SharedPreferences or files**
- Use **Android Keystore** as the key provider — keys never leave the secure hardware
- **EncryptedSharedPreferences** for key-value sensitive data (tokens, settings)
- **EncryptedFile** for sensitive file content
- **SQLCipher** for encrypted Room databases
- Encryption keys are **generated once and stored in Keystore** — never hardcoded

---

## EncryptedSharedPreferences

```kotlin
// ✅ Create encrypted SharedPreferences
fun createEncryptedPreferences(context: Context): SharedPreferences {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    return EncryptedSharedPreferences.create(
        context,
        "secure_prefs",                                      // file name
        masterKey,
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
    )
}

// ✅ Wrapper for type-safe access
class SecurePreferences @Inject constructor(
    @ApplicationContext context: Context
) {
    private val prefs = createEncryptedPreferences(context)

    var authToken: String?
        get() = prefs.getString(KEY_AUTH_TOKEN, null)
        set(value) = prefs.edit {
            if (value != null) putString(KEY_AUTH_TOKEN, value)
            else remove(KEY_AUTH_TOKEN)
        }

    var refreshToken: String?
        get() = prefs.getString(KEY_REFRESH_TOKEN, null)
        set(value) = prefs.edit {
            if (value != null) putString(KEY_REFRESH_TOKEN, value)
            else remove(KEY_REFRESH_TOKEN)
        }

    fun clearAll() = prefs.edit { clear() }

    companion object {
        private const val KEY_AUTH_TOKEN = "auth_token"
        private const val KEY_REFRESH_TOKEN = "refresh_token"
    }
}
```

---

## EncryptedFile

```kotlin
// ✅ Write encrypted file
fun writeEncryptedFile(context: Context, fileName: String, content: ByteArray) {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    val file = File(context.filesDir, fileName)
    if (file.exists()) file.delete()   // EncryptedFile cannot overwrite

    val encryptedFile = EncryptedFile.Builder(
        context,
        file,
        masterKey,
        EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
    ).build()

    encryptedFile.openFileOutput().use { output ->
        output.write(content)
    }
}

// ✅ Read encrypted file
fun readEncryptedFile(context: Context, fileName: String): ByteArray {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    val file = File(context.filesDir, fileName)
    val encryptedFile = EncryptedFile.Builder(
        context,
        file,
        masterKey,
        EncryptedFile.FileEncryptionScheme.AES256_GCM_HKDF_4KB
    ).build()

    return encryptedFile.openFileInput().use { it.readBytes() }
}
```

---

## Encrypted DataStore

```kotlin
// ✅ Encrypt DataStore using EncryptedFile as backing storage
// Add dependency: androidx.security:security-crypto-ktx

fun createEncryptedDataStore(context: Context): DataStore<Preferences> {
    val masterKey = MasterKey.Builder(context)
        .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
        .build()

    return PreferenceDataStoreFactory.createWithPath(
        produceFile = {
            val file = File(context.filesDir, "secure_datastore.preferences_pb")
            // For true file-level encryption, wrap with EncryptedFile
            file.toOkioPath()
        }
    )
}

// ✅ Simpler: use Tink-based DataStore encryption
// (preferred for production)
class EncryptedDataStoreManager @Inject constructor(
    private val dataStore: DataStore<Preferences>
) {
    private val TOKEN_KEY = stringPreferencesKey("auth_token")
    private val USER_ID_KEY = stringPreferencesKey("user_id")

    val authToken: Flow<String?> = dataStore.data
        .catch { if (it is IOException) emit(emptyPreferences()) else throw it }
        .map { it[TOKEN_KEY] }

    suspend fun saveAuthToken(token: String) {
        dataStore.edit { it[TOKEN_KEY] = token }
    }

    suspend fun clearAuth() {
        dataStore.edit {
            it.remove(TOKEN_KEY)
            it.remove(USER_ID_KEY)
        }
    }
}
```

---

## Encrypted Room (SQLCipher)

```kotlin
// ✅ SQLCipher for encrypted Room database
// dependency: net.zetetic:android-database-sqlcipher
// dependency: androidx.sqlite:sqlite-ktx

@Database(entities = [...], version = 1)
abstract class SecureDatabase : RoomDatabase() {
    abstract fun sensitiveDataDao(): SensitiveDataDao
}

fun buildSecureDatabase(context: Context, passphrase: ByteArray): SecureDatabase {
    val factory = SupportFactory(passphrase)
    return Room.databaseBuilder(context, SecureDatabase::class.java, "secure.db")
        .openHelperFactory(factory)
        .build()
}

// ✅ Derive passphrase from Keystore-backed key
class DatabaseKeyProvider @Inject constructor(
    @ApplicationContext private val context: Context
) {
    fun getPassphrase(): ByteArray {
        val keyAlias = "db_key"
        val keyStore = KeyStore.getInstance("AndroidKeyStore").also { it.load(null) }

        if (!keyStore.containsAlias(keyAlias)) {
            val keyGenerator = KeyGenerator.getInstance(
                KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
            )
            keyGenerator.init(
                KeyGenParameterSpec.Builder(
                    keyAlias,
                    KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
                )
                    .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                    .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                    .setKeySize(256)
                    .build()
            )
            keyGenerator.generateKey()
        }

        // Derive a stable passphrase from the key
        val key = keyStore.getKey(keyAlias, null) as SecretKey
        return key.encoded ?: ByteArray(32).also { SecureRandom().nextBytes(it) }
    }
}
```

---

## Hilt Setup

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object StorageModule {

    @Provides
    @Singleton
    fun provideSecurePreferences(
        @ApplicationContext context: Context
    ): SecurePreferences = SecurePreferences(context)

    @Provides
    @Singleton
    fun provideSecureDatabase(
        @ApplicationContext context: Context,
        keyProvider: DatabaseKeyProvider
    ): SecureDatabase = buildSecureDatabase(context, keyProvider.getPassphrase())
}
```

---

## What to Encrypt

| Data | Storage | Encrypted? |
|---|---|---|
| Auth token / refresh token | EncryptedSharedPreferences | ✅ Always |
| User PII (name, email, phone) | Encrypted Room / SQLCipher | ✅ Always |
| Payment data | Encrypted Room | ✅ Always |
| App preferences (theme, language) | Regular DataStore | ❌ Not needed |
| Cached images | File cache | ❌ Usually not needed |
| Health/medical data | Encrypted Room | ✅ Always |

---

## Anti-Patterns

- Storing tokens in plain `SharedPreferences` — readable by rooted devices and backup extraction
- Hardcoding encryption keys in source code — use Android Keystore
- Storing the encryption key alongside the encrypted data — defeats the purpose
- Using `MODE_WORLD_READABLE` or `MODE_WORLD_WRITEABLE` for any file
- Encrypting non-sensitive data — unnecessary overhead; encrypt only what needs protection

---

## Related Skills
- `keystore` — Android Keystore key generation and management
- `datastore` — DataStore patterns
- `room` — Room database setup
- `encrypted-database` — full SQLCipher Room setup
- `secure-networking` — protecting data in transit
