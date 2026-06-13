---
name: hook-detection
description: >
  Runtime hook detection for Android apps.
  Load this skill when detecting Xposed framework, LSPosed, or other
  hooking frameworks that intercept method calls, alter app behavior,
  or bypass security checks at runtime.
---

# Hook Detection

## Overview
Hooking frameworks like Xposed and LSPosed allow attackers to intercept and modify method calls at runtime — bypassing authentication, logging sensitive data, or altering app behavior. Hook detection identifies the presence of these frameworks before they can tamper with critical operations.

---

## Core Principles

- **Detect early** — check before any sensitive operation, not just at startup
- **Multiple signals** — Xposed/LSPosed can hide some traces but rarely all
- **Combine with root detection** — hooking frameworks typically require root
- **Server-side validation** — client-side detection can be bypassed; pair with Play Integrity
- **Fail-secure** — block sensitive operations if hooks are detected

---

## Xposed / LSPosed Detection

```kotlin
class HookDetector @Inject constructor(
    @ApplicationContext private val context: Context
) {
    data class HookCheckResult(
        val isHooked: Boolean,
        val signals: List<String>
    )

    fun check(): HookCheckResult {
        val signals = mutableListOf<String>()

        if (checkXposedInstaller()) signals.add("xposed_installer")
        if (checkXposedBridge()) signals.add("xposed_bridge")
        if (checkStackTrace()) signals.add("xposed_stack_trace")
        if (checkXposedFiles()) signals.add("xposed_files")
        if (checkLoadedModules()) signals.add("loaded_modules")

        return HookCheckResult(
            isHooked = signals.isNotEmpty(),
            signals = signals
        )
    }

    // ✅ Check for Xposed installer apps
    private fun checkXposedInstaller(): Boolean {
        val xposedPackages = listOf(
            "de.robv.android.xposed.installer",
            "org.meowcat.edxposed.manager",
            "io.github.lsposed.manager",
            "com.android.xposed"
        )
        return xposedPackages.any { pkg ->
            try {
                context.packageManager.getPackageInfo(pkg, 0)
                true
            } catch (e: PackageManager.NameNotFoundException) {
                false
            }
        }
    }

    // ✅ Check if XposedBridge class is loadable
    private fun checkXposedBridge(): Boolean {
        return try {
            Class.forName("de.robv.android.xposed.XposedBridge")
            true
        } catch (e: ClassNotFoundException) {
            false
        }
    }

    // ✅ Check stack trace for Xposed method wrappers
    private fun checkStackTrace(): Boolean {
        return try {
            throw Exception("stack_check")
        } catch (e: Exception) {
            val stackTrace = e.stackTrace.joinToString { it.className }
            stackTrace.contains("XposedBridge") ||
            stackTrace.contains("de.robv.android.xposed")
        }
    }

    // ✅ Check for Xposed-related files on disk
    private fun checkXposedFiles(): Boolean {
        val xposedFiles = listOf(
            "/system/lib/libxposed_art.so",
            "/system/lib64/libxposed_art.so",
            "/system/xposed.prop",
            "/data/data/de.robv.android.xposed.installer"
        )
        return xposedFiles.any { File(it).exists() }
    }

    // ✅ Check loaded libraries for hooking framework signatures
    private fun checkLoadedModules(): Boolean {
        return try {
            val mapsFile = File("/proc/self/maps")
            mapsFile.readLines().any { line ->
                line.contains("XposedBridge") ||
                line.contains("libxposed") ||
                line.contains("edxp") ||
                line.contains("lspd")
            }
        } catch (e: Exception) {
            false
        }
    }
}
```

---

## Runtime Integrity Check

```kotlin
// ✅ Verify critical method hasn't been hooked by checking its declaring class
fun isMethodHooked(methodName: String, clazz: Class<*>): Boolean {
    return try {
        val method = clazz.getDeclaredMethod(methodName)
        // If method is hooked, its class will be XposedBridge's handler class
        method.declaringClass != clazz
    } catch (e: Exception) {
        false
    }
}

// ✅ Self-check on a known method
fun checkOwnMethodIntegrity(): Boolean {
    return try {
        val method = MySecurityClass::class.java.getDeclaredMethod("checkOwnMethodIntegrity")
        method.declaringClass == MySecurityClass::class.java
    } catch (e: Exception) {
        false
    }
}
```

---

## Response Strategy

```kotlin
@HiltViewModel
class SecurityViewModel @Inject constructor(
    private val hookDetector: HookDetector,
    private val rootDetector: RootDetector
) : ViewModel() {

    val securityStatus: StateFlow<SecurityStatus> = flow {
        val hookResult = hookDetector.check()
        val rootResult = rootDetector.check()

        emit(SecurityStatus(
            isHooked = hookResult.isHooked,
            isRooted = rootResult.isRooted,
            hookSignals = hookResult.signals,
            rootSignals = rootResult.signals
        ))
    }.stateIn(viewModelScope, SharingStarted.Eagerly, SecurityStatus())
}

data class SecurityStatus(
    val isHooked: Boolean = false,
    val isRooted: Boolean = false,
    val hookSignals: List<String> = emptyList(),
    val rootSignals: List<String> = emptyList()
) {
    val isCompromised: Boolean get() = isHooked || isRooted
}
```

---

## Anti-Patterns

- Only checking for Xposed installer — LSPosed and EdXposed use different package names
- Not checking at runtime before sensitive ops — startup-only check can be bypassed
- Crashing without explanation — show a clear message before blocking
- Trusting client-side detection alone — combine with Play Integrity API for server validation
- Storing detection result in a predictable variable — hooked code can modify it

---

## Related Skills
- `frida-detection` — Frida-specific detection
- `root-detection` — detecting rooted devices
- `reverse-engineering-resistance` — broader anti-tampering measures
- `obfuscation` — making detection logic harder to analyze
