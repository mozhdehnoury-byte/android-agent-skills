---
name: reverse-engineering-resistance
description: >
  Reverse engineering resistance for Android apps.
  Load this skill when designing a defense-in-depth security strategy,
  combining multiple anti-tampering measures, protecting sensitive algorithms,
  implementing APK integrity checks, or hardening an app against static analysis.
---

# Reverse Engineering Resistance

## Overview
Reverse engineering resistance is not a single technique — it's a layered strategy. Attackers use static analysis (decompiling the APK), dynamic analysis (runtime hooks, Frida), and debugging. No single measure stops a determined attacker, but raising the cost and complexity of an attack is a meaningful defense.

---

## Core Principles

- **Defense in depth** — layer multiple techniques; no single measure is sufficient
- **Detect and respond** — detect analysis attempts and fail-secure
- **Move secrets server-side** — anything in the APK can be extracted
- **Obfuscate detection logic** — easy-to-read detection code is easy to bypass
- **Server-side integrity** — Play Integrity API provides the strongest client attestation

---

## Defense Layers

```
Layer 1: Build-time protection
  → R8 obfuscation + shrinking
  → String encryption
  → Resource obfuscation

Layer 2: Runtime integrity checks
  → APK signature verification
  → Debugger detection
  → Emulator detection
  → Root detection
  → Hook detection (Xposed, Frida)

Layer 3: Server-side attestation
  → Play Integrity API
  → Certificate pinning
  → Short-lived tokens
```

---

## APK Signature Verification

```kotlin
// ✅ Verify the app is signed with the expected certificate
class ApkIntegrityChecker @Inject constructor(
    @ApplicationContext private val context: Context
) {
    // Get your signing cert hash:
    // keytool -printcert -file CERT.RSA | grep SHA256
    private val expectedCertHash = "AA:BB:CC:DD:..."   // replace with actual hash

    fun isSignatureValid(): Boolean {
        return try {
            val packageInfo = context.packageManager.getPackageInfo(
                context.packageName,
                PackageManager.GET_SIGNING_CERTIFICATES
            )
            val signingInfo = packageInfo.signingInfo
            val signatures = signingInfo?.apkContentsSigners ?: return false

            signatures.any { signature ->
                val digest = MessageDigest.getInstance("SHA-256")
                val certHash = digest.digest(signature.toByteArray())
                val hashHex = certHash.joinToString(":") { "%02X".format(it) }
                hashHex == expectedCertHash
            }
        } catch (e: Exception) {
            false
        }
    }
}
```

---

## Debugger Detection

```kotlin
// ✅ Detect if a debugger is attached
object DebuggerDetector {
    fun isDebuggerAttached(): Boolean =
        android.os.Debug.isDebuggerConnected() ||
        android.os.Debug.waitingForDebugger()

    fun isDebuggableBuild(): Boolean =
        BuildConfig.DEBUG ||
        (android.os.Debug.isDebuggerConnected())

    // ✅ Check android:debuggable in manifest at runtime
    fun isManifestDebuggable(context: Context): Boolean {
        val appInfo = context.applicationInfo
        return (appInfo.flags and android.content.pm.ApplicationInfo.FLAG_DEBUGGABLE) != 0
    }
}
```

---

## Emulator Detection

```kotlin
// ✅ Detect common Android emulators
object EmulatorDetector {
    fun isEmulator(): Boolean =
        checkBuildProperties() || checkEmulatorFiles() || checkTelephony()

    private fun checkBuildProperties(): Boolean {
        val indicators = listOf(
            Build.FINGERPRINT.startsWith("generic"),
            Build.FINGERPRINT.startsWith("unknown"),
            Build.MODEL.contains("google_sdk", ignoreCase = true),
            Build.MODEL.contains("Emulator", ignoreCase = true),
            Build.MODEL.contains("Android SDK built for x86", ignoreCase = true),
            Build.MANUFACTURER.contains("Genymotion", ignoreCase = true),
            Build.HARDWARE.contains("goldfish", ignoreCase = true),
            Build.HARDWARE.contains("ranchu", ignoreCase = true),
            Build.PRODUCT.contains("sdk_gphone", ignoreCase = true),
            Build.PRODUCT.contains("vbox86p", ignoreCase = true)
        )
        return indicators.any { it }
    }

    private fun checkEmulatorFiles(): Boolean {
        val emulatorFiles = listOf(
            "/dev/socket/qemud",
            "/dev/qemu_pipe",
            "/system/lib/libc_malloc_debug_qemu.so",
            "/sys/qemu_trace",
            "/system/bin/qemu-props"
        )
        return emulatorFiles.any { File(it).exists() }
    }

    private fun checkTelephony(): Boolean {
        // IMEI "000000000000000" is used by many emulators
        return Build.SERIAL == "unknown" || Build.SERIAL == "0"
    }
}
```

---

## String Obfuscation Pattern

```kotlin
// ✅ Don't store sensitive strings as plain constants
// ❌ Bad — visible in decompiled APK
private const val API_SECRET = "my_secret_key_12345"

// ✅ Better — obfuscate with simple XOR (raise the bar, not a crypto solution)
object StringObfuscator {
    private val key = byteArrayOf(0x4A, 0x3F, 0x2E, 0x1D, 0x5C)

    fun decode(encoded: ByteArray): String {
        return String(ByteArray(encoded.size) { i ->
            (encoded[i].toInt() xor key[i % key.size].toInt()).toByte()
        })
    }
}

// ✅ Best — keep secrets server-side entirely; fetch at runtime with auth
```

---

## Tamper Response Strategy

```kotlin
// ✅ Consolidated tamper check at app startup
class TamperDetectionService @Inject constructor(
    private val apkIntegrityChecker: ApkIntegrityChecker,
    private val rootDetector: RootDetector,
    private val hookDetector: HookDetector,
    private val fridaDetector: FridaDetector
) {
    sealed interface TamperStatus {
        data object Clean : TamperStatus
        data class Compromised(val reasons: List<String>) : TamperStatus
    }

    suspend fun evaluate(): TamperStatus = withContext(Dispatchers.IO) {
        val reasons = mutableListOf<String>()

        if (!apkIntegrityChecker.isSignatureValid()) reasons.add("invalid_signature")
        if (DebuggerDetector.isDebuggerAttached()) reasons.add("debugger_attached")
        if (EmulatorDetector.isEmulator()) reasons.add("emulator")
        if (rootDetector.check().isRooted) reasons.add("rooted")
        if (hookDetector.check().isHooked) reasons.add("hooked")
        if (fridaDetector.check().isDetected) reasons.add("frida")

        if (reasons.isEmpty()) TamperStatus.Clean
        else TamperStatus.Compromised(reasons)
    }
}

// ✅ Response options per risk level
fun respondToTamper(status: TamperStatus, sensitivity: AppSensitivity) {
    if (status is TamperStatus.Compromised) {
        when (sensitivity) {
            AppSensitivity.HIGH -> {
                // Banking, health: terminate
                showTamperDialog()
                finishAndRemoveTask()
            }
            AppSensitivity.MEDIUM -> {
                // Restrict sensitive features only
                disableSensitiveFeatures()
                logThreatToServer(status.reasons)
            }
            AppSensitivity.LOW -> {
                // Log silently
                logThreatToServer(status.reasons)
            }
        }
    }
}
```

---

## Play Integrity API (Strongest Attestation)

```kotlin
// ✅ Use Play Integrity for server-verified device integrity
// The server receives and verifies a signed token from Google
// Token contains: appIntegrity, deviceIntegrity, accountDetails

suspend fun requestIntegrityToken(context: Context, nonce: String): Result<String> =
    runCatching {
        val manager = IntegrityManagerFactory.create(context)
        val request = IntegrityTokenRequest.builder().setNonce(nonce).build()
        manager.requestIntegrityToken(request).await().token()
    }

// Server checks:
// deviceRecognitionVerdict: MEETS_STRONG_INTEGRITY → real unmodified device
// appIntegrityVerdict: PLAY_RECOGNIZED → APK matches Play store version
```

---

## Anti-Patterns

- Relying on obfuscation alone — determined attackers can re-analyze obfuscated code
- Storing API secrets in APK — use server-side secrets with authenticated access
- Android `debuggable=true` in release builds — allows trivial debugging and memory inspection
- Single-point detection — check at startup only; also check before each sensitive operation
- No server-side validation — all client-side checks are bypassable given enough effort

---

## Related Skills
- `root-detection` — rooted device detection
- `hook-detection` — Xposed/LSPosed detection
- `frida-detection` — Frida dynamic instrumentation detection
- `obfuscation` — R8 obfuscation configuration
- `certificate-pinning` — network-level protection
- `cryptography` — protecting sensitive data
