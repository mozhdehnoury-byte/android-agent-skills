---
name: biometric
description: >
  Biometric authentication for Android apps.
  Load this skill when implementing fingerprint or face authentication,
  using BiometricPrompt, tying biometric auth to Keystore keys,
  handling fallback to device credentials, or checking biometric availability.
---

# Biometric

## Overview
Android's BiometricPrompt API provides a unified interface for fingerprint, face, and iris authentication. Biometric auth can be used standalone (gate access to a screen) or tied to a Keystore key (unlock a cryptographic operation). The latter is stronger — authentication is required each time the key is used.

---

## Core Principles

- Use **BiometricPrompt** — never use FingerprintManager directly (deprecated)
- **Crypto-based auth** (tied to Keystore key) is stronger than confirmation-only auth
- Always provide **fallback to device credentials** (PIN/pattern/password) for accessibility
- Handle `KeyPermanentlyInvalidatedException` — thrown when new biometrics are enrolled
- Check availability before showing biometric UI — degrade gracefully on unsupported devices

---

## Check Availability

```kotlin
// ✅ Check if biometric is available before using it
class BiometricAvailabilityChecker @Inject constructor(
    @ApplicationContext private val context: Context
) {
    sealed interface Availability {
        data object Available : Availability
        data object NotEnrolled : Availability
        data object NotSupported : Availability
        data object HardwareUnavailable : Availability
    }

    fun check(): Availability {
        val manager = BiometricManager.from(context)
        return when (manager.canAuthenticate(
            BiometricManager.Authenticators.BIOMETRIC_STRONG or
            BiometricManager.Authenticators.DEVICE_CREDENTIAL
        )) {
            BiometricManager.BIOMETRIC_SUCCESS          -> Availability.Available
            BiometricManager.BIOMETRIC_ERROR_NONE_ENROLLED -> Availability.NotEnrolled
            BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE   -> Availability.NotSupported
            BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> Availability.HardwareUnavailable
            else -> Availability.NotSupported
        }
    }
}
```

---

## Simple Biometric Prompt (Confirmation Only)

```kotlin
// ✅ Gate screen access — no Keystore key involved
class BiometricAuthManager @Inject constructor() {

    sealed interface AuthResult {
        data object Success : AuthResult
        data class Error(val message: String) : AuthResult
        data object Cancelled : AuthResult
    }

    fun authenticate(
        activity: FragmentActivity,
        title: String = "Authenticate",
        subtitle: String = "Use biometric or device credential",
        onResult: (AuthResult) -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)

        val prompt = BiometricPrompt(activity, executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    onResult(AuthResult.Success)
                }
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    if (errorCode == BiometricPrompt.ERROR_USER_CANCELED ||
                        errorCode == BiometricPrompt.ERROR_NEGATIVE_BUTTON) {
                        onResult(AuthResult.Cancelled)
                    } else {
                        onResult(AuthResult.Error(errString.toString()))
                    }
                }
                override fun onAuthenticationFailed() {
                    // Called on each failed attempt — not terminal; don't handle as final failure
                }
            }
        )

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle(title)
            .setSubtitle(subtitle)
            .setAllowedAuthenticators(
                BiometricManager.Authenticators.BIOMETRIC_STRONG or
                BiometricManager.Authenticators.DEVICE_CREDENTIAL
            )
            .build()

        prompt.authenticate(promptInfo)
    }
}
```

---

## Crypto-Based Biometric (Keystore-Tied)

```kotlin
// ✅ Biometric auth that unlocks a Keystore key — stronger guarantee
class BiometricCryptoManager @Inject constructor(
    private val keystoreCrypto: KeystoreCrypto
) {
    private val keyAlias = KeystoreAliases.BIOMETRIC_KEY

    // ✅ Prepare cipher before showing prompt — pass to authenticate()
    fun prepareCipher(): Cipher? {
        return try {
            val keyStore = KeyStore.getInstance("AndroidKeyStore").also { it.load(null) }
            if (!keyStore.containsAlias(keyAlias)) generateBiometricBoundKey(keyAlias)

            val key = keyStore.getKey(keyAlias, null) as SecretKey
            Cipher.getInstance("AES/GCM/NoPadding").also {
                it.init(Cipher.ENCRYPT_MODE, key)
            }
        } catch (e: KeyPermanentlyInvalidatedException) {
            // New biometrics enrolled — delete and recreate key
            KeyStore.getInstance("AndroidKeyStore").also { it.load(null) }.deleteEntry(keyAlias)
            null
        }
    }

    fun authenticate(
        activity: FragmentActivity,
        cipher: Cipher,
        onSuccess: (Cipher) -> Unit,
        onFailure: () -> Unit
    ) {
        val executor = ContextCompat.getMainExecutor(activity)
        val cryptoObject = BiometricPrompt.CryptoObject(cipher)

        val prompt = BiometricPrompt(activity, executor,
            object : BiometricPrompt.AuthenticationCallback() {
                override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
                    result.cryptoObject?.cipher?.let { onSuccess(it) } ?: onFailure()
                }
                override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
                    onFailure()
                }
            }
        )

        val promptInfo = BiometricPrompt.PromptInfo.Builder()
            .setTitle("Confirm identity")
            .setSubtitle("Touch sensor to continue")
            .setAllowedAuthenticators(BiometricManager.Authenticators.BIOMETRIC_STRONG)
            .setNegativeButtonText("Cancel")
            .build()

        prompt.authenticate(promptInfo, cryptoObject)
    }
}
```

---

## Compose Integration

```kotlin
// ✅ Trigger biometric from Compose via Activity
@Composable
fun BiometricGatedScreen(
    onAuthenticated: () -> Unit,
    content: @Composable () -> Unit
) {
    val activity = LocalContext.current as FragmentActivity
    val biometricManager = remember { BiometricAuthManager() }
    var isAuthenticated by rememberSaveable { mutableStateOf(false) }

    LaunchedEffect(Unit) {
        biometricManager.authenticate(
            activity = activity,
            onResult = { result ->
                if (result is BiometricAuthManager.AuthResult.Success) {
                    isAuthenticated = true
                    onAuthenticated()
                }
            }
        )
    }

    if (isAuthenticated) {
        content()
    } else {
        Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
        }
    }
}
```

---

## Anti-Patterns

- Using `FingerprintManager` — deprecated since API 28; use `BiometricPrompt`
- Not handling `KeyPermanentlyInvalidatedException` — crashes when user enrolls new fingerprint
- No fallback to device credential — locks out users with biometric issues
- Calling `authenticate()` from a background thread — must be called on main thread
- Storing sensitive data in plain storage after biometric success — biometric should unlock Keystore key

---

## Related Skills
- `keystore` — generating biometric-bound Keystore keys
- `encrypted-storage` — protecting data unlocked by biometric
- `cryptography` — cipher operations after biometric success
