---
name: logging
description: >
  Logging strategy and implementation for Android apps.
  Load this skill when setting up Timber, defining log levels,
  controlling log output per build type, tagging logs consistently,
  or deciding what should and should not be logged.
---

# Logging

## Overview
Logging is the first line of observability in Android. Timber is the standard logging library — it wraps Android's `Log` class, adds automatic tag detection, and supports pluggable trees for different build types. In debug builds, logs go to Logcat. In release builds, logs go to Crashlytics or a remote logging service.

---

## Core Principles

- Use **Timber** — never `Log.d()` directly in application code
- Never log **sensitive data** — tokens, passwords, PII, payment info
- Log at the **right level** — don't use `d` for errors or `e` for debug info
- Debug logs are **free in debug builds** — don't hold back useful context
- Release builds should send errors and warnings to a remote service, not Logcat

---

## Setup

```toml
# libs.versions.toml
[libraries]
timber = { module = "com.jakewharton.timber:timber", version = "5.0.1" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.timber)
}
```

---

## Initializing Timber

```kotlin
// ✅ Plant trees in Application.onCreate()
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())  // Logcat in debug
        } else {
            Timber.plant(CrashlyticsTree())   // Remote in release
        }
    }
}

// ✅ Release tree — sends warnings and errors to Crashlytics
class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.WARN) return  // skip DEBUG and INFO in release

        when (priority) {
            Log.WARN  -> FirebaseCrashlytics.getInstance().log("W/$tag: $message")
            Log.ERROR -> {
                FirebaseCrashlytics.getInstance().log("E/$tag: $message")
                t?.let { FirebaseCrashlytics.getInstance().recordException(it) }
            }
        }
    }
}
```

---

## Log Levels

```kotlin
// ✅ VERBOSE — highly detailed, development only
Timber.v("Received ${users.size} users from cache")

// ✅ DEBUG — useful development info
Timber.d("Loading user profile for id=$userId")

// ✅ INFO — significant lifecycle events
Timber.i("User logged in: userId=$userId")

// ✅ WARN — unexpected but recoverable situation
Timber.w("Cache miss for key=$key, falling back to network")

// ✅ ERROR — something failed, needs attention
Timber.e(exception, "Failed to sync user data")

// ✅ WTF — should never happen, treat as fatal
Timber.wtf("Received null userId in authenticated state")
```

---

## Tagging

```kotlin
// ✅ Timber auto-detects class name as tag — no manual tagging needed
class UserRepository {
    fun loadUser() {
        Timber.d("Loading user")  // tag = "UserRepository" automatically
    }
}

// ✅ Manual tag when needed (e.g. in utility functions)
Timber.tag("NetworkMonitor").d("Connection state: $state")
```

---

## What NOT to Log

```kotlin
// ❌ Never log sensitive data
Timber.d("Token: $accessToken")           // security risk
Timber.d("Password entered: $password")   // security risk
Timber.d("Card number: $cardNumber")      // PCI violation

// ✅ Log safe identifiers only
Timber.d("User authenticated: userId=${userId.take(4)}***")
Timber.d("Token refreshed successfully")
```

---

## Logging in Repositories and Use Cases

```kotlin
// ✅ Log at entry and error points
class UserRepositoryImpl : UserRepository {

    override suspend fun getUser(id: String): Result<User> {
        Timber.d("Fetching user: id=$id")

        return runCatching {
            api.getUser(id).also {
                Timber.d("User fetched successfully: id=$id")
            }
        }.onFailure { error ->
            Timber.e(error, "Failed to fetch user: id=$id")
        }.map { mapper.toDomain(it) }
    }
}
```

---

## Anti-Patterns

- Using `Log.d()` directly — bypasses tree system, always prints in release
- Logging sensitive data (tokens, PII, passwords) — security and privacy violation
- Using `Timber.e()` for expected errors (empty state, 404) — use `Timber.w()` instead
- No logging in release builds — blind to production issues
- Over-logging — logging every line makes logs unreadable

---

## Related Skills
- `structured-logging` — structured log format for log aggregation
- `crash-reporting` — integrating logs with crash reports
- `observability` — broader observability strategy
