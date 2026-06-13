---
name: kmp
description: >
  Kotlin Multiplatform (KMP) setup, structure, and best practices.
  Load this skill when setting up a KMP project, sharing code between
  Android and other platforms, or deciding what belongs in shared vs
  platform-specific modules.
---

# Kotlin Multiplatform (KMP)

## Overview

KMP allows sharing Kotlin code across platforms (Android, iOS, Desktop, Web) while keeping platform-specific implementations separate. The goal is to maximize shared logic while respecting platform boundaries.

---

## Core Principles

- Share **business logic** — not UI, not platform APIs
- Use `expect/actual` only when necessary — prefer interface + DI
- Keep shared code free of any platform dependency
- The `commonMain` module must never import Android or iOS specific APIs

---

## Module Structure

```
project/
├── shared/
│   ├── src/
│   │   ├── commonMain/kotlin/     ← shared business logic
│   │   ├── commonTest/kotlin/     ← shared unit tests
│   │   ├── androidMain/kotlin/    ← Android actual implementations
│   │   └── iosMain/kotlin/        ← iOS actual implementations
├── androidApp/                    ← Android UI & entry point
└── iosApp/                        ← iOS UI & entry point
```

---

## What Goes Where

| Layer                      | commonMain | androidMain / iosMain |
| -------------------------- | ---------- | --------------------- |
| Domain models              | ✅          | ❌                     |
| Use cases                  | ✅          | ❌                     |
| Repository interfaces      | ✅          | ❌                     |
| Repository implementations | ❌          | ✅                     |
| Network (Ktor)             | ✅          | ❌                     |
| Database (SQLDelight)      | ✅ schema   | ✅ driver              |
| UI                         | ❌          | ✅                     |
| Platform APIs              | ❌          | ✅                     |

---

## expect / actual Pattern

### Prefer interface + DI over expect/actual

```kotlin
// ✅ Preferred — interface in commonMain, implementation injected
interface PlatformLogger {
    fun log(message: String)
}

// androidMain
class AndroidLogger : PlatformLogger {
    override fun log(message: String) = Log.d("App", message)
}
```

### Use expect/actual only for simple platform utilities

```kotlin
// commonMain
expect fun getCurrentTimeMillis(): Long

// androidMain
actual fun getCurrentTimeMillis(): Long = System.currentTimeMillis()

// iosMain
actual fun getCurrentTimeMillis(): Long =
    NSDate().timeIntervalSince1970.toLong() * 1000
```

---

## Networking — Ktor (shared)

```kotlin
// commonMain — use Ktor, not Retrofit (Retrofit is Android-only)
val client = HttpClient {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true })
    }
    install(HttpTimeout) {
        requestTimeoutMillis = 30_000
    }
}
```

## Database — SQLDelight (shared schema)

```kotlin
// commonMain — define schema in .sq files
// androidMain — provide AndroidSqliteDriver
// iosMain    — provide NativeSqliteDriver
```

---

## Coroutines in KMP

```kotlin
// ✅ Use kotlinx.coroutines in commonMain
// ✅ Use Dispatchers.Default for CPU work in common code
// ✅ Use platform dispatcher injection for UI thread

// androidMain
val uiDispatcher: CoroutineDispatcher = Dispatchers.Main

// iosMain
val uiDispatcher: CoroutineDispatcher = Dispatchers.Main
```

---

## Gradle Setup (libs.versions.toml)

```toml
[versions]
kotlin = "2.0.0"
kmp = "2.0.0"

[plugins]
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
```

```kotlin
// shared/build.gradle.kts
kotlin {
    androidTarget()
    iosX64()
    iosArm64()
    iosSimulatorArm64()

    sourceSets {
        commonMain.dependencies {
            implementation(libs.ktor.client.core)
            implementation(libs.kotlinx.coroutines.core)
            implementation(libs.kotlinx.serialization.json)
        }
        androidMain.dependencies {
            implementation(libs.ktor.client.okhttp)
        }
        iosMain.dependencies {
            implementation(libs.ktor.client.darwin)
        }
    }
}
```

---

## Anti-Patterns

- Importing `android.*` in `commonMain` — breaks iOS compilation
- Using Retrofit in shared code — it's Android-only; use Ktor
- Putting UI logic in `commonMain` — UI must stay platform-specific
- Overusing `expect/actual` — prefer interface + DI for testability
- Sharing ViewModels in commonMain — keep ViewModels platform-specific

---

## Related Skills

- `kotlin` — Kotlin language conventions
- `coroutine` — coroutine patterns in shared and platform code
- `serialization` — Kotlinx Serialization setup for KMP
- `ktor` — Ktor client setup and usage
