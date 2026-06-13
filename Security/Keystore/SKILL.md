---
name: keystore
description: >
  Android Keystore System for cryptographic key management.
  Load this skill when generating, storing, or using cryptographic keys,
  implementing encrypt/decrypt operations backed by Keystore,
  requiring user authentication before key use, or managing key lifecycle.
---

# Keystore

## Overview
The Android Keystore System lets you store cryptographic keys in a container that makes them harder to extract — keys can be bound to secure hardware (TEE/StrongBox) and never leave the device. Keystore is the foundation for all local encryption, biometric authentication, and signing operations.

---

## Core Principles

- Keys are **generated inside Keystore** — never imported from outside unless necessary
- Keys **never leave the secure enclave** — only the operations (encrypt/decrypt/sign) are performed
- Specify **purpose at key generation** — a key for encryption cannot be used for signing
- Use **StrongBox** when available for highest security (hardware security module)
- Keys can require **user authentication** (biometric/PIN) before use

---

## Generating a Symmetric Key (AES)

```kotlin
// ✅ Generate AES key for encryption/decryption
fun generateAesKey(alias: String) {
    val keyGenerator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES,
        "AndroidKeyStore"
    )

    keyGenerator.init(
        KeyGenParameterSpec.Builder(
            alias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            .setUserAuthenticationRequired(false)   // set true to require biometric/PIN
            .setInvalidatedByBiometricEnrollment(true)
            .build()
    )

    keyGenerator.generateKey()
}

// ✅ Generate key that requires biometric authentication
fun generateBiometricBoundKey(alias: String) {
    val keyGenerator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES,
        "AndroidKeyStore"
    )

    keyGenerator.init(
        KeyGenParameterSpec.Builder(
            alias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256)
            .setUserAuthenticationRequired(true)
            .setUserAuthenticationParameters(
                0,   // 0 = every use requires auth
                KeyProperties.AUTH_BIOMETRIC_STRONG or KeyProperties.AUTH_DEVICE_CREDENTIAL
            )
            .setInvalidatedByBiometricEnrollment(true)
            .build()
    )

    keyGenerator.generateKey()
}
```

---

## Generating an Asymmetric Key (RSA / EC)

```kotlin
// ✅ Generate EC key pair for signing
fun generateEcKeyPair(alias: String) {
    val keyPairGenerator = KeyPairGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_EC,
        "AndroidKeyStore"
    )

    keyPairGenerator.initialize(
        KeyGenParameterSpec.Builder(
            alias,
            KeyProperties.PURPOSE_SIGN or KeyProperties.PURPOSE_VERIFY
        )
            .setAlgorithmParameterSpec(ECGenParameterSpec("secp256r1"))
            .setDigests(KeyProperties.DIGEST_SHA256, KeyProperties.DIGEST_SHA512)
            .build()
    )

    keyPairGenerator.generateKeyPair()
}

// ✅ Generate RSA key pair for encryption
fun generateRsaKeyPair(alias: String) {
    val keyPairGenerator = KeyPairGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_RSA,
        "AndroidKeyStore"
    )

    keyPairGenerator.initialize(
        KeyGenParameterSpec.Builder(
            alias,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setKeySize(2048)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_RSA_OAEP)
            .setDigests(KeyProperties.DIGEST_SHA256)
            .build()
    )

    keyPairGenerator.generateKeyPair()
}
```

---

## Encrypt / Decrypt with AES-GCM

```kotlin
class KeystoreCrypto @Inject constructor() {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").also { it.load(null) }

    // ✅ Encrypt — returns IV + ciphertext (IV needed for decryption)
    fun encrypt(alias: String, plaintext: ByteArray): EncryptedData {
        ensureKeyExists(alias)
        val key = keyStore.getKey(alias, null) as SecretKey
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        cipher.init(Cipher.ENCRYPT_MODE, key)

        val ciphertext = cipher.doFinal(plaintext)
        return EncryptedData(iv = cipher.iv, ciphertext = ciphertext)
    }

    // ✅ Decrypt — requires same IV used during encryption
    fun decrypt(alias: String, data: EncryptedData): ByteArray {
        val key = keyStore.getKey(alias, null) as SecretKey
        val cipher = Cipher.getInstance("AES/GCM/NoPadding")
        val spec = GCMParameterSpec(128, data.iv)
        cipher.init(Cipher.DECRYPT_MODE, key, spec)
        return cipher.doFinal(data.ciphertext)
    }

    fun deleteKey(alias: String) {
        if (keyStore.containsAlias(alias)) {
            keyStore.deleteEntry(alias)
        }
    }

    fun hasKey(alias: String): Boolean = keyStore.containsAlias(alias)

    private fun ensureKeyExists(alias: String) {
        if (!keyStore.containsAlias(alias)) generateAesKey(alias)
    }
}

data class EncryptedData(
    val iv: ByteArray,
    val ciphertext: ByteArray
) {
    // ✅ Serialize for storage — prepend IV to ciphertext
    fun toByteArray(): ByteArray = iv + ciphertext

    companion object {
        private const val IV_SIZE = 12  // GCM IV is always 12 bytes

        fun fromByteArray(data: ByteArray): EncryptedData = EncryptedData(
            iv = data.copyOfRange(0, IV_SIZE),
            ciphertext = data.copyOfRange(IV_SIZE, data.size)
        )
    }
}
```

---

## Sign / Verify with EC

```kotlin
class KeystoreSigner @Inject constructor() {
    private val keyStore = KeyStore.getInstance("AndroidKeyStore").also { it.load(null) }

    fun sign(alias: String, data: ByteArray): ByteArray {
        val privateKey = keyStore.getKey(alias, null) as PrivateKey
        return Signature.getInstance("SHA256withECDSA").run {
            initSign(privateKey)
            update(data)
            sign()
        }
    }

    fun verify(alias: String, data: ByteArray, signature: ByteArray): Boolean {
        val publicKey = keyStore.getCertificate(alias).publicKey
        return Signature.getInstance("SHA256withECDSA").run {
            initVerify(publicKey)
            update(data)
            verify(signature)
        }
    }
}
```

---

## StrongBox (Hardware Security Module)

```kotlin
// ✅ Use StrongBox when available (Pixel 3+, Android 9+)
fun generateStrongBoxKey(context: Context, alias: String) {
    val hasStrongBox = context.packageManager
        .hasSystemFeature(PackageManager.FEATURE_STRONGBOX_KEYSTORE)

    val spec = KeyGenParameterSpec.Builder(
        alias,
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
    )
        .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
        .setKeySize(256)
        .apply {
            if (hasStrongBox) setIsStrongBoxBacked(true)
        }
        .build()

    KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        .apply { init(spec) }
        .generateKey()
}
```

---

## Key Aliases (Convention)

```kotlin
object KeystoreAliases {
    const val DATABASE_KEY   = "app_database_key"
    const val AUTH_TOKEN_KEY = "app_auth_token_key"
    const val BIOMETRIC_KEY  = "app_biometric_key"
    const val SIGNING_KEY    = "app_signing_key"
}
```

---

## Anti-Patterns

- Generating keys outside AndroidKeyStore — keys in memory can be extracted
- Reusing the same IV for multiple encryptions with the same key — breaks GCM security
- Storing the IV separately in an insecure location — IV must be stored with ciphertext
- Using ECB mode (`AES/ECB/PKCS5Padding`) — no IV, patterns visible in ciphertext
- Ignoring `KeyPermanentlyInvalidatedException` — thrown when biometrics change; handle gracefully

---

## Related Skills
- `encrypted-storage` — using Keystore-backed keys for storage encryption
- `biometric` — biometric authentication tied to Keystore keys
- `cryptography` — higher-level crypto operations
- `secure-networking` — certificate pinning and TLS
