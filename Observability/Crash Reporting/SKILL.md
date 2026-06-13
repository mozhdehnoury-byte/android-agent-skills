---
name: crash-reporting
description: >
  Crash reporting setup and best practices for Android using Firebase Crashlytics.
  Load this skill when configuring Crashlytics, adding custom context to crash
  reports, handling non-fatal errors, filtering noise, or integrating crash
  reporting with the logging pipeline.
---

# Crash Reporting

## Overview
Crash reporting captures uncaught exceptions and ANRs in production, providing stack traces, device info, and custom context. Firebase Crashlytics is the standard for Android. The goal is to have high signal-to-noise ratio — every reported issue should be actionable.

---

## Core Principles

- Report **non-fatal errors** explicitly — don't wait for crashes to find bugs
- Add **custom context** before crashes happen — user state, feature flags, last actions
- Set **user ID** on login/logout — essential for understanding crash impact
- Filter out **known non-actionable** crashes — third-party SDK crashes, OOM on old devices
- Test crash reporting in debug with a deliberate crash — don't assume it works

---

## Setup

```toml
# libs.versions.toml
[plugins]
google-services = { id = "com.google.gms.google-services", version = "4.4.2" }
firebase-crashlytics = { id = "com.google.firebase.crashlytics", version = "3.0.2" }

[libraries]
firebase-crashlytics = { module = "com.google.firebase:firebase-crashlytics-ktx" }
firebase-bom = { module = "com.google.firebase:firebase-bom", version = "33.1.0" }
```

```kotlin
// app/build.gradle.kts
plugins {
    alias(libs.plugins.google.services)
    alias(libs.plugins.firebase.crashlytics)
}

dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.crashlytics)
}
```

---

## Initialization

```kotlin
// ✅ Configure in Application.onCreate()
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        FirebaseCrashlytics.getInstance().apply {
            // Disable in debug to avoid noise in the dashboard
            setCrashlyticsCollectionEnabled(!BuildConfig.DEBUG)
        }
    }
}
```

---

## Setting User Context

```kotlin
// ✅ Set user ID and properties on login
class AuthManager @Inject constructor(
    private val crashlytics: FirebaseCrashlytics
) {
    fun onUserLoggedIn(user: User) {
        crashlytics.setUserId(user.id)
        crashlytics.setCustomKey("user_plan", user.plan)
        crashlytics.setCustomKey("user_region", user.region)
    }

    fun onUserLoggedOut() {
        crashlytics.setUserId("")
        crashlytics.setCustomKey("user_plan", "none")
    }
}
```

---

## Custom Keys for Context

```kotlin
// ✅ Set keys before operations that might crash
class SyncManager @Inject constructor(
    private val crashlytics: FirebaseCrashlytics
) {
    suspend fun sync(userId: String) {
        // Set context before the risky operation
        crashlytics.setCustomKey("sync_user_id", userId)
        crashlytics.setCustomKey("sync_started_at", System.currentTimeMillis())
        crashlytics.setCustomKey("sync_attempt", syncAttemptCount)

        try {
            performSync(userId)
            crashlytics.setCustomKey("sync_status", "success")
        } catch (e: Exception) {
            crashlytics.setCustomKey("sync_status", "failed")
            throw e
        }
    }
}
```

---

## Non-Fatal Error Reporting

```kotlin
// ✅ Record non-fatal errors for bugs that don't crash but are wrong
class UserRepository @Inject constructor(
    private val crashlytics: FirebaseCrashlytics
) {
    suspend fun getUser(id: String): Result<User> {
        return runCatching { api.getUser(id) }
            .onFailure { error ->
                when (error) {
                    is HttpException -> {
                        if (error.code() == 500) {
                            // Server error — report as non-fatal
                            crashlytics.recordException(error)
                        }
                        // 4xx — expected, don't report
                    }
                    is IOException -> {
                        // Network error — expected, don't report
                    }
                    else -> {
                        // Unexpected error — always report
                        crashlytics.recordException(error)
                    }
                }
            }
            .map { mapper.toDomain(it) }
    }
}
```

---

## Breadcrumbs via Log

```kotlin
// ✅ Add breadcrumbs — recent actions shown in crash report
class NavigationLogger @Inject constructor(
    private val crashlytics: FirebaseCrashlytics
) {
    fun onNavigate(route: String) {
        // Crashlytics.log() messages appear as breadcrumbs in crash reports
        crashlytics.log("Navigate to: $route")
    }
}

// ✅ Log important user actions as breadcrumbs
crashlytics.log("User tapped delete on item: id=$itemId")
crashlytics.log("Sync started: attempt=$attempt")
crashlytics.log("Payment flow entered")
```

---

## Timber Integration

```kotlin
// ✅ CrashlyticsTree — route Timber errors to Crashlytics
class CrashlyticsTree : Timber.Tree() {
    private val crashlytics = FirebaseCrashlytics.getInstance()

    override fun log(priority: Int, tag: String?, message: String, t: Throwable?) {
        crashlytics.log("${priorityLabel(priority)}/$tag: $message")

        if (priority == Log.ERROR) {
            t?.let { crashlytics.recordException(it) }
                ?: crashlytics.recordException(Exception("Error logged without exception: $message"))
        }
    }

    private fun priorityLabel(priority: Int) = when (priority) {
        Log.VERBOSE -> "V"
        Log.DEBUG   -> "D"
        Log.INFO    -> "I"
        Log.WARN    -> "W"
        Log.ERROR   -> "E"
        else        -> "?"
    }
}
```

---

## Anti-Patterns

- Enabling Crashlytics in debug builds — floods dashboard with non-production crashes
- Not setting user context — can't measure crash impact by user count
- Reporting all `IOException` as non-fatal — network errors are expected, creates noise
- Not using `setCustomKey` before risky operations — crash report lacks context
- Catching all exceptions silently without reporting — bugs hidden in production

---

## Related Skills
- `logging` — Timber setup that feeds into Crashlytics
- `structured-logging` — adding structured context to crash reports
- `metrics` — tracking error rates alongside crashes
- `firebase` — Firebase setup and configuration
