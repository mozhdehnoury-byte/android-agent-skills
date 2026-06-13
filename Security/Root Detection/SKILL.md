---
name: root-detection
description: >
  Root detection for Android security hardening.
  Load this skill when detecting rooted devices, implementing multi-signal
  root checks, deciding how to respond to rooted devices, or combining
  root detection with other integrity checks.
---

# Root Detection

## Overview
Root detection identifies devices where the Android security model has been compromised. A rooted device can bypass app-level protections, read encrypted storage, hook function calls, and tamper with app behavior. Root detection is one layer in a defense-in-depth strategy — no single check is foolproof.

---

## Core Principles

- **Multiple signals** — no single check is reliable; combine several
- **Fail-secure** — if detection is uncertain, treat as rooted in high-security contexts
- **Graceful degradation** — decide per-feature whether to block, warn, or restrict
- **Obfuscate detection logic** — easy-to-read detection code is easy to bypass
- **Server-side validation** — combine with Play Integrity API for stronger guarantees

---

## Multi-Signal Root Detection

```kotlin
class RootDetector @Inject constructor(
    @ApplicationContext private val context: Context
) {
    data class RootCheckResult(
        val isRooted: Boolean,
        val signals: List<String>   // which signals triggered
    )

    fun check(): RootCheckResult {
        val signals = mutableListOf<String>()

        if (checkSuBinary()) signals.add("su_binary")
        if (checkRootApps()) signals.add("root_apps")
        if (checkDangerousProps()) signals.add("dangerous_props")
        if (checkRwPaths()) signals.add("rw_system_paths")
        if (checkTestKeys()) signals.add("test_keys")
        if (checkMagiskFiles()) signals.add("magisk_files")

        return RootCheckResult(
            isRooted = signals.isNotEmpty(),
            signals = signals
        )
    }

    // ✅ Check for su binary in common locations
    private fun checkSuBinary(): Boolean {
        val paths = listOf(
            "/system/bin/su", "/system/xbin/su", "/sbin/su",
            "/system/su", "/system/bin/.ext/.su", "/system/usr/we-need-root/su-backup",
            "/data/local/su", "/data/local/bin/su", "/data/local/xbin/su"
        )
        return paths.any { File(it).exists() }
    }

    // ✅ Check for known root management apps
    private fun checkRootApps(): Boolean {
        val rootPackages = listOf(
            "com.topjohnwu.magisk",
            "com.noshufou.android.su",
            "com.koushikdutta.superuser",
            "eu.chainfire.supersu",
            "com.kingroot.kinguser",
            "com.kingo.root",
            "com.smedialink.oneclickroot"
        )
        return rootPackages.any { pkg ->
            try {
                context.packageManager.getPackageInfo(pkg, 0)
                true
            } catch (e: PackageManager.NameNotFoundException) {
                false
            }
        }
    }

    // ✅ Check dangerous build properties
    private fun checkDangerousProps(): Boolean {
        val dangerousProps = mapOf(
            "ro.debuggable" to "1",
            "ro.secure" to "0",
            "ro.build.type" to "eng"
        )
        return dangerousProps.any { (prop, dangerousValue) ->
            readSystemProperty(prop) == dangerousValue
        }
    }

    // ✅ Check if /system is writable (should always be read-only)
    private fun checkRwPaths(): Boolean {
        val rwPaths = listOf("/system", "/system/bin", "/system/sbin", "/system/xbin", "/vendor/bin")
        return rwPaths.any { path ->
            try {
                val file = File(path)
                file.exists() && file.canWrite()
            } catch (e: Exception) {
                false
            }
        }
    }

    // ✅ Check for test-keys build (non-official signing)
    private fun checkTestKeys(): Boolean {
        val buildTags = Build.TAGS
        return buildTags != null && buildTags.contains("test-keys")
    }

    // ✅ Check for Magisk-specific files
    private fun checkMagiskFiles(): Boolean {
        val magiskPaths = listOf(
            "/sbin/.magisk", "/sbin/.core/mirror",
            "/sbin/.core/img", "/data/adb/magisk"
        )
        return magiskPaths.any { File(it).exists() }
    }

    private fun readSystemProperty(name: String): String? = try {
        val process = Runtime.getRuntime().exec(arrayOf("getprop", name))
        process.inputStream.bufferedReader().readLine()?.trim()
    } catch (e: Exception) {
        null
    }
}
```

---

## Play Integrity API (Recommended for Production)

```kotlin
// ✅ Play Integrity API — server-side verification, harder to bypass
class IntegrityChecker @Inject constructor(
    @ApplicationContext private val context: Context
) {
    suspend fun checkIntegrity(nonce: String): Result<String> = runCatching {
        val integrityManager = IntegrityManagerFactory.create(context)
        val request = IntegrityTokenRequest.builder()
            .setNonce(nonce)
            .build()

        val response = integrityManager.requestIntegrityToken(request).await()
        response.token()   // send this token to your server for verification
    }
}

// Server verifies the token via Google Play Integrity API
// Token contains: deviceIntegrity, appIntegrity, accountDetails
// deviceIntegrity.deviceRecognitionVerdict includes MEETS_STRONG_INTEGRITY
```

---

## Response Strategy

```kotlin
// ✅ Decide response per security level
@HiltViewModel
class SecurityViewModel @Inject constructor(
    private val rootDetector: RootDetector
) : ViewModel() {

    sealed interface SecurityState {
        data object Safe : SecurityState
        data class Compromised(val signals: List<String>) : SecurityState
    }

    val securityState: StateFlow<SecurityState> = flow {
        val result = rootDetector.check()
        emit(
            if (result.isRooted) SecurityState.Compromised(result.signals)
            else SecurityState.Safe
        )
    }.stateIn(viewModelScope, SharingStarted.Eagerly, SecurityState.Safe)
}

// ✅ UI response options (choose based on app sensitivity)
// Option 1: Block entirely (banking, health apps)
if (securityState is SecurityState.Compromised) {
    RootedDeviceBlockScreen()
    return
}

// Option 2: Warn and restrict features
if (securityState is SecurityState.Compromised) {
    showWarning("Some features are unavailable on rooted devices")
    disableSensitiveFeatures()
}

// Option 3: Log and monitor (lower security apps)
if (securityState is SecurityState.Compromised) {
    analytics.logEvent("rooted_device_detected", securityState.signals)
}
```

---

## Anti-Patterns

- Relying on a single check — root tools specifically bypass known single checks
- Blocking without user explanation — frustrates legitimate users (developers, testers)
- Performing root check on main thread — run on IO dispatcher
- Storing root check result in plain SharedPreferences — a rooted device can modify it
- Skipping server-side validation for high-security operations — client checks are bypassable

---

## Related Skills
- `hook-detection` — detecting Frida and Xposed hooks
- `frida-detection` — Frida-specific detection techniques
- `reverse-engineering-resistance` — broader anti-tampering measures
- `obfuscation` — making detection logic harder to analyze
- `certificate-pinning` — network-level security complement
