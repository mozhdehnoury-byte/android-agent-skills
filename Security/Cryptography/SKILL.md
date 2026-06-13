---
name: cryptography
description: >
  Cryptographic operations for Android apps.
  Load this skill when implementing hashing, symmetric encryption,
  asymmetric encryption, digital signatures, key derivation,
  HMAC, or secure random generation.
---

# Cryptography

## Overview
Android provides the standard Java Cryptography Architecture (JCA) with hardware-backed key storage via Keystore. This skill covers the correct algorithms and patterns for common cryptographic needs — encryption, hashing, signing, and key derivation — without relying on Keystore (see `keystore` skill for that).

---

## Core Principles

- **Use proven algorithms** — AES-GCM, ChaCha20-Poly1305, RSA-OAEP, SHA-256, PBKDF2
- **Never roll your own crypto** — use standard JCA or Tink
- **Always use authenticated encryption** (AES-GCM, not AES-CBC without MAC)
- **Never reuse IV/nonce** with the same key
- **Use Tink for high-level operations** — fewer footguns than raw JCA

---

## Symmetric Encryption (AES-GCM)

```kotlin
object AesGcm {
    private const val ALGORITHM = "AES/GCM/NoPadding"
    private const val KEY_SIZE = 256
    private const val IV_SIZE = 12      // 96 bits — standard for GCM
    private const val TAG_SIZE = 128    // authentication tag bits

    // ✅ Generate a random AES key (for non-Keystore use)
    fun generateKey(): SecretKey {
        val generator = KeyGenerator.getInstance("AES")
        generator.init(KEY_SIZE)
        return generator.generateKey()
    }

    // ✅ Encrypt — returns IV prepended to ciphertext
    fun encrypt(key: SecretKey, plaintext: ByteArray): ByteArray {
        val iv = ByteArray(IV_SIZE).also { SecureRandom().nextBytes(it) }
        val cipher = Cipher.getInstance(ALGORITHM)
        cipher.init(Cipher.ENCRYPT_MODE, key, GCMParameterSpec(TAG_SIZE, iv))
        val ciphertext = cipher.doFinal(plaintext)
        return iv + ciphertext   // prepend IV for storage
    }

    // ✅ Decrypt — expects IV prepended to ciphertext
    fun decrypt(key: SecretKey, data: ByteArray): ByteArray {
        val iv = data.copyOfRange(0, IV_SIZE)
        val ciphertext = data.copyOfRange(IV_SIZE, data.size)
        val cipher = Cipher.getInstance(ALGORITHM)
        cipher.init(Cipher.DECRYPT_MODE, key, GCMParameterSpec(TAG_SIZE, iv))
        return cipher.doFinal(ciphertext)
    }
}
```

---

## Hashing

```kotlin
object Hashing {
    // ✅ SHA-256 hash
    fun sha256(input: ByteArray): ByteArray =
        MessageDigest.getInstance("SHA-256").digest(input)

    fun sha256Hex(input: String): String =
        sha256(input.toByteArray(Charsets.UTF_8))
            .joinToString("") { "%02x".format(it) }

    // ✅ SHA-256 of a file (streaming — no OOM for large files)
    fun sha256File(file: File): ByteArray {
        val digest = MessageDigest.getInstance("SHA-256")
        file.inputStream().use { stream ->
            val buffer = ByteArray(8192)
            var read: Int
            while (stream.read(buffer).also { read = it } != -1) {
                digest.update(buffer, 0, read)
            }
        }
        return digest.digest()
    }

    // ✅ HMAC-SHA256 — for message authentication
    fun hmacSha256(key: ByteArray, message: ByteArray): ByteArray {
        val secretKey = SecretKeySpec(key, "HmacSHA256")
        return Mac.getInstance("HmacSHA256").run {
            init(secretKey)
            doFinal(message)
        }
    }
}
```

---

## Password Hashing (PBKDF2)

```kotlin
// ✅ Hash passwords for local verification — NOT for sending to server
object PasswordHasher {
    private const val ITERATIONS = 100_000
    private const val KEY_LENGTH = 256
    private const val SALT_SIZE = 16
    private const val ALGORITHM = "PBKDF2WithHmacSHA256"

    data class HashedPassword(val hash: ByteArray, val salt: ByteArray) {
        fun toStorageString(): String =
            "${Base64.encodeToString(hash, Base64.NO_WRAP)}:${Base64.encodeToString(salt, Base64.NO_WRAP)}"

        companion object {
            fun fromStorageString(value: String): HashedPassword {
                val (hashB64, saltB64) = value.split(":")
                return HashedPassword(
                    hash = Base64.decode(hashB64, Base64.NO_WRAP),
                    salt = Base64.decode(saltB64, Base64.NO_WRAP)
                )
            }
        }
    }

    fun hash(password: CharArray): HashedPassword {
        val salt = ByteArray(SALT_SIZE).also { SecureRandom().nextBytes(it) }
        val spec = PBEKeySpec(password, salt, ITERATIONS, KEY_LENGTH)
        val factory = SecretKeyFactory.getInstance(ALGORITHM)
        val hash = factory.generateSecret(spec).encoded
        spec.clearPassword()
        return HashedPassword(hash, salt)
    }

    fun verify(password: CharArray, stored: HashedPassword): Boolean {
        val spec = PBEKeySpec(password, stored.salt, ITERATIONS, KEY_LENGTH)
        val factory = SecretKeyFactory.getInstance(ALGORITHM)
        val hash = factory.generateSecret(spec).encoded
        spec.clearPassword()
        return MessageDigest.isEqual(hash, stored.hash)
    }
}
```

---

## Key Derivation (HKDF)

```kotlin
// ✅ Derive multiple keys from one master key
object KeyDerivation {
    fun deriveKey(masterKey: ByteArray, info: String, outputLength: Int = 32): ByteArray {
        // HKDF using HMAC-SHA256
        val salt = ByteArray(32)   // zero salt
        val prk = Hashing.hmacSha256(salt, masterKey)
        val infoBytes = info.toByteArray(Charsets.UTF_8)
        val okm = Hashing.hmacSha256(prk, infoBytes + byteArrayOf(1))
        return okm.copyOfRange(0, outputLength)
    }
}

// Usage: derive separate keys for encryption and authentication from one master
val encryptionKey = KeyDerivation.deriveKey(masterKey, "encryption")
val authKey = KeyDerivation.deriveKey(masterKey, "authentication")
```

---

## Secure Random

```kotlin
// ✅ Always use SecureRandom for cryptographic purposes
object SecureRandomUtils {
    fun randomBytes(size: Int): ByteArray =
        ByteArray(size).also { SecureRandom().nextBytes(it) }

    fun randomHex(size: Int): String =
        randomBytes(size).joinToString("") { "%02x".format(it) }

    fun randomBase64(size: Int): String =
        Base64.encodeToString(randomBytes(size), Base64.NO_WRAP)
}
```

---

## Tink (Recommended for Production)

```kotlin
// ✅ Google Tink — high-level crypto library, fewer footguns
// dependency: com.google.crypto.tink:tink-android

// Initialize once at app startup
TinkConfig.register()

// ✅ Authenticated encryption
val keysetHandle = KeysetHandle.generateNew(AeadKeyTemplates.AES256_GCM)
val aead = keysetHandle.getPrimitive(Aead::class.java)
val ciphertext = aead.encrypt(plaintext, associatedData)
val decrypted = aead.decrypt(ciphertext, associatedData)

// ✅ Store keysets securely with Keystore integration
val keysetHandle = AndroidKeysetManager.Builder()
    .withSharedPref(context, "tink_keyset", "tink_prefs")
    .withKeyTemplate(AeadKeyTemplates.AES256_GCM)
    .withMasterKeyUri("android-keystore://tink_master_key")
    .build()
    .keysetHandle
```

---

## Algorithm Selection Guide

| Need | Algorithm | Avoid |
|---|---|---|
| Symmetric encryption | AES-256-GCM | AES-ECB, AES-CBC without MAC |
| Password hashing | PBKDF2 (100k+ iterations) | MD5, SHA-1, unsalted hash |
| General hashing | SHA-256 / SHA-3 | MD5, SHA-1 |
| Message authentication | HMAC-SHA256 | Custom MAC schemes |
| Asymmetric encryption | RSA-OAEP-SHA256 | RSA-PKCS1v1.5 |
| Digital signature | ECDSA P-256, Ed25519 | RSA with SHA-1 |
| Key agreement | ECDH P-256 | DH without authentication |

---

## Anti-Patterns

- `Random()` instead of `SecureRandom()` — predictable, not suitable for crypto
- AES-ECB mode — same plaintext block → same ciphertext; patterns visible
- Reusing IV with the same key in GCM — catastrophic key recovery attack
- MD5 or SHA-1 for security purposes — cryptographically broken
- Storing keys as hex strings in SharedPreferences — use Android Keystore
- Rolling custom encryption schemes — use standard JCA or Tink

---

## Related Skills
- `keystore` — hardware-backed key storage for Android
- `encrypted-storage` — applying crypto to local storage
- `biometric` — biometric-gated crypto operations
- `secure-networking` — TLS and certificate pinning
