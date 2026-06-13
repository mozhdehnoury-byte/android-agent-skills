---
name: crashlytics
description: >
  Firebase Crashlytics setup and usage for Android crash reporting.
  Load this skill when configuring Crashlytics, logging custom events,
  adding user context to crash reports, or handling non-fatal errors.
---

# Crashlytics

## Overview
Firebase Crashlytics is a real-time crash reporter that helps track, prioritize, and fix stability issues. It automatically captures fatal crashes and provides detailed stack traces, device info, and custom context. Non-fatal errors can also be reported manually.

---

## Core Principles

- **Disable** Crashlytics in debug builds — don't pollute production data with dev crashes
- Add **user context** before crashes happen — userId, screen, and key attributes help diagnose
- Report **non-fatal errors** explicitly — caught exceptions that indicate problems
- Use **custom keys** to add state context — what the user was doing before the crash
- Always pair Crashlytics with **proper logging** — crash reports alone miss context

---

## Setup

```kotlin
// build.gradle.kts (app)
plugins {
    alias(libs.plugins.firebase.crashlytics)
}

dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.crashlytics)
}
```

---

## Enable / Disable by Build Type

```kotlin
// ✅ Disable in debug — only report in release
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        FirebaseCrashlytics.getInstance()
            .setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
    }
}
```

---

## User Context

```kotlin
// ✅ Set user identity — helps group crashes by user
class CrashlyticsUserTracker @Inject constructor() {

    fun setUser(userId: String, email: String?) {
        FirebaseCrashlytics.getInstance().apply {
            setUserId(userId)
            email?.let { setCustomKey("user_email", it) }
        }
    }

    fun clearUser() {
        FirebaseCrashlytics.getInstance().apply {
            setUserId("")
            setCustomKey("user_email", "")
        }
    }
}
```

---

## Custom Keys — State Context

```kotlin
// ✅ Log app state before a crash
class CrashlyticsContext @Inject constructor() {

    fun setScreen(screenName: String) {
        FirebaseCrashlytics.getInstance()
            .setCustomKey("current_screen", screenName)
    }

    fun setFeatureFlag(key: String, value: Boolean) {
        FirebaseCrashlytics.getInstance()
            .setCustomKey("flag_$key", value)
    }

    fun setApiEndpoint(endpoint: String) {
        FirebaseCrashlytics.getInstance()
            .setCustomKey("last_api_endpoint", endpoint)
    }

    fun setDatabaseVersion(version: Int) {
        FirebaseCrashlytics.getInstance()
            .setCustomKey("db_version", version)
    }
}
```

---

## Breadcrumb Logging

```kotlin
// ✅ Log events leading up to a crash
class CrashlyticsLogger @Inject constructor() {

    fun log(message: String) {
        FirebaseCrashlytics.getInstance().log(message)
    }

    fun logUserAction(action: String) {
        FirebaseCrashlytics.getInstance().log("USER_ACTION: $action")
    }

    fun logNetworkCall(endpoint: String, statusCode: Int) {
        FirebaseCrashlytics.getInstance()
            .log("NETWORK: $endpoint -> $statusCode")
    }
}
```

---

## Non-Fatal Error Reporting

```kotlin
// ✅ Report caught exceptions that indicate a problem
class UserRepository @Inject constructor(
    private val crashlytics: CrashlyticsLogger
) {
    suspend fun getUser(id: String): Result<User> {
        return try {
            val user = api.getUser(id)
            Result.success(user.toDomain())
        } catch (e: HttpException) {
            if (e.code() == 500) {
                // ✅ Server errors are worth tracking
                FirebaseCrashlytics.getInstance().recordException(e)
            }
            Result.failure(e)
        } catch (e: Exception) {
            // ✅ Unexpected exceptions always reported
            FirebaseCrashlytics.getInstance().recordException(e)
            Result.failure(e)
        }
    }
}

// ✅ In global error handler
class GlobalExceptionHandler @Inject constructor() {

    fun report(throwable: Throwable, context: String) {
        FirebaseCrashlytics.getInstance().apply {
            setCustomKey("error_context", context)
            recordException(throwable)
        }
    }
}
```

---

## Integration with Timber

```kotlin
// ✅ Route Timber ERROR logs to Crashlytics
class CrashlyticsTree : Timber.Tree() {
    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        if (priority < Log.ERROR) return

        FirebaseCrashlytics.getInstance().apply {
            tag?.let { setCustomKey("log_tag", it) }
            log(message)
            t?.let { recordException(it) }
        }
    }
}

// Plant in Application
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        } else {
            Timber.plant(CrashlyticsTree())
        }
    }
}
```

---

## Testing Crashes

```kotlin
// ✅ Force a test crash (debug only)
if (BuildConfig.DEBUG) {
    Button(onClick = {
        FirebaseCrashlytics.getInstance().log("Test crash triggered")
        throw RuntimeException("Test crash")
    }) {
        Text("Test Crash")
    }
}
```

---

## Anti-Patterns

- Enabling Crashlytics in debug builds — pollutes crash data with dev noise
- No user context — impossible to reproduce user-specific crashes
- Reporting every caught exception — noise drowns out real issues; be selective
- No custom keys — crash report without state context is hard to diagnose
- Not logging breadcrumbs — no trail of events leading to the crash

---

## Related Skills
- `firebase` — Firebase core setup
- `analytics` — event tracking alongside crash reporting
- `logging` — structured logging complementing Crashlytics
- `error-handling` — error handling policy across layers
