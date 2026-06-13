---
name: frida-detection
description: >
  Frida dynamic instrumentation detection for Android apps.
  Load this skill when detecting Frida gadget injection, Frida server
  presence, or dynamic instrumentation attempts targeting the app at runtime.
---

# Frida Detection

## Overview
Frida is a dynamic instrumentation toolkit widely used by security researchers and attackers to hook functions, inspect memory, and bypass security checks at runtime. Unlike Xposed (which requires a framework installed on the device), Frida can be injected into a process dynamically. Detecting it requires checking for its artifacts at multiple levels.

---

## Core Principles

- **Frida leaves multiple traces** — process maps, port, libraries, named pipes
- **Check at multiple points** — startup and before sensitive operations
- **Frida can hide itself** — combine with server-side integrity (Play Integrity API)
- **Native checks are harder to bypass** — JNI-based detection is more resilient than Java
- **No detection is perfect** — the goal is to raise the cost of attack

---

## Java-Level Detection

```kotlin
class FridaDetector @Inject constructor() {

    data class FridaCheckResult(
        val isDetected: Boolean,
        val signals: List<String>
    )

    fun check(): FridaCheckResult {
        val signals = mutableListOf<String>()

        if (checkFridaServer()) signals.add("frida_server_port")
        if (checkFridaFiles()) signals.add("frida_files")
        if (checkProcessMaps()) signals.add("frida_in_maps")
        if (checkFridaNamedPipes()) signals.add("frida_named_pipes")
        if (checkFridaLibraries()) signals.add("frida_libraries")

        return FridaCheckResult(
            isDetected = signals.isNotEmpty(),
            signals = signals
        )
    }

    // ✅ Check if Frida server is listening on its default port (27042)
    private fun checkFridaServer(): Boolean {
        return try {
            Socket().use { socket ->
                socket.connect(InetSocketAddress("127.0.0.1", 27042), 100)
                true   // connection succeeded — Frida server likely running
            }
        } catch (e: Exception) {
            false
        }
    }

    // ✅ Check for Frida-related files
    private fun checkFridaFiles(): Boolean {
        val fridaPaths = listOf(
            "/data/local/tmp/frida-server",
            "/data/local/tmp/frida",
            "/sdcard/frida-server",
            "/system/bin/frida-server"
        )
        return fridaPaths.any { File(it).exists() }
    }

    // ✅ Check /proc/self/maps for Frida agent
    private fun checkProcessMaps(): Boolean {
        return try {
            File("/proc/self/maps").readLines().any { line ->
                line.contains("frida", ignoreCase = true) ||
                line.contains("gum-js-loop", ignoreCase = true) ||
                line.contains("gmain", ignoreCase = true) ||
                line.contains("linjector", ignoreCase = true)
            }
        } catch (e: Exception) {
            false
        }
    }

    // ✅ Check for Frida named pipes in /proc/self/fd
    private fun checkFridaNamedPipes(): Boolean {
        return try {
            val fdDir = File("/proc/self/fd")
            fdDir.listFiles()?.any { fd ->
                try {
                    val link = fd.canonicalPath
                    link.contains("frida", ignoreCase = true) ||
                    link.contains("linjector", ignoreCase = true)
                } catch (e: Exception) {
                    false
                }
            } ?: false
        } catch (e: Exception) {
            false
        }
    }

    // ✅ Check loaded libraries for Frida signature
    private fun checkFridaLibraries(): Boolean {
        return try {
            File("/proc/self/maps").readLines().any { line ->
                line.contains("frida-agent", ignoreCase = true) ||
                line.contains("frida-gadget", ignoreCase = true)
            }
        } catch (e: Exception) {
            false
        }
    }
}
```

---

## Native Detection (JNI — More Resilient)

```c
// ✅ native-lib.cpp — harder to hook than Java methods
#include <jni.h>
#include <string>
#include <fstream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

extern "C" JNIEXPORT jboolean JNICALL
Java_com_example_app_security_FridaDetectorNative_checkFridaPort(JNIEnv*, jobject) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) return JNI_FALSE;

    struct sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(27042);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);

    struct timeval tv{};
    tv.tv_sec = 0;
    tv.tv_usec = 100000;  // 100ms timeout
    setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
    setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv));

    bool detected = connect(sock, (struct sockaddr*)&addr, sizeof(addr)) == 0;
    close(sock);
    return detected ? JNI_TRUE : JNI_FALSE;
}

extern "C" JNIEXPORT jboolean JNICALL
Java_com_example_app_security_FridaDetectorNative_checkProcessMaps(JNIEnv*, jobject) {
    std::ifstream maps("/proc/self/maps");
    std::string line;
    while (std::getline(maps, line)) {
        if (line.find("frida") != std::string::npos ||
            line.find("gum-js-loop") != std::string::npos ||
            line.find("frida-agent") != std::string::npos) {
            return JNI_TRUE;
        }
    }
    return JNI_FALSE;
}
```

```kotlin
// ✅ Kotlin wrapper for native checks
class FridaDetectorNative {
    external fun checkFridaPort(): Boolean
    external fun checkProcessMaps(): Boolean

    companion object {
        init { System.loadLibrary("native-lib") }
    }
}
```

---

## Combined Security Check

```kotlin
// ✅ Unified check combining all detectors
class AppSecurityChecker @Inject constructor(
    private val fridaDetector: FridaDetector,
    private val hookDetector: HookDetector,
    private val rootDetector: RootDetector
) {
    data class SecurityReport(
        val passed: Boolean,
        val threats: List<String>
    )

    suspend fun runChecks(): SecurityReport = withContext(Dispatchers.IO) {
        val threats = mutableListOf<String>()

        val frida = fridaDetector.check()
        val hooks = hookDetector.check()
        val root = rootDetector.check()

        if (frida.isDetected) threats.addAll(frida.signals.map { "frida:$it" })
        if (hooks.isHooked)   threats.addAll(hooks.signals.map { "hook:$it" })
        if (root.isRooted)    threats.addAll(root.signals.map { "root:$it" })

        SecurityReport(passed = threats.isEmpty(), threats = threats)
    }
}
```

---

## Anti-Patterns

- Only checking port 27042 — Frida server can be configured to use any port
- Java-only detection — Frida can hook Java methods; native checks are more reliable
- Checking only at startup — Frida can be attached after the app starts
- Crashing the app on detection without explanation — show a clear message
- No server-side corroboration — client detection alone is bypassable

---

## Related Skills
- `hook-detection` — Xposed/LSPosed framework detection
- `root-detection` — rooted device detection
- `reverse-engineering-resistance` — broader anti-tampering strategy
- `obfuscation` — obfuscating detection logic
