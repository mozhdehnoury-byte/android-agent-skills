---
name: app-startup
description: >
  Jetpack App Startup library for initializer management in Android.
  Load this skill when controlling SDK initialization order, lazy-loading
  initializers, replacing ContentProvider-based auto-init, or reducing
  the number of ContentProviders at startup.
---

# App Startup

## Overview
The Jetpack App Startup library provides a standard way to initialize components at app startup. It consolidates multiple `ContentProvider` initializers (used by many SDKs) into a single one, reducing startup overhead. It also supports lazy initialization and explicit dependency ordering between initializers.

---

## Core Principles

- Use App Startup to **consolidate** multiple SDK ContentProviders into one
- Define **dependencies** between initializers to control order
- Use **lazy initialization** for SDKs not needed immediately
- Disable auto-init for SDKs that have their own ContentProvider when using App Startup
- Keep each `Initializer.create()` fast — it runs on the main thread during startup

---

## Setup

```toml
# libs.versions.toml
[libraries]
androidx-startup = { module = "androidx.startup:startup-runtime", version = "1.1.1" }
```

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.androidx.startup)
}
```

---

## Creating Initializers

```kotlin
// ✅ Timber initializer
class TimberInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        if (BuildConfig.DEBUG) {
            Timber.plant(Timber.DebugTree())
        }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// ✅ Analytics initializer — depends on Timber being initialized first
class AnalyticsInitializer : Initializer<AnalyticsClient> {
    override fun create(context: Context): AnalyticsClient {
        return AnalyticsClient.initialize(context).also {
            Timber.d("Analytics initialized")
        }
    }

    override fun dependencies(): List<Class<out Initializer<*>>> =
        listOf(TimberInitializer::class.java)  // ✅ Timber initializes first
}

// ✅ WorkManager initializer
class WorkManagerInitializer : Initializer<WorkManager> {
    override fun create(context: Context): WorkManager {
        val config = Configuration.Builder()
            .setMinimumLoggingLevel(Log.INFO)
            .build()
        WorkManager.initialize(context, config)
        return WorkManager.getInstance(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}
```

---

## Manifest Registration

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false">

    <!-- Register initializers -->
    <meta-data
        android:name="com.example.TimberInitializer"
        android:value="androidx.startup" />

    <meta-data
        android:name="com.example.AnalyticsInitializer"
        android:value="androidx.startup" />

    <meta-data
        android:name="com.example.WorkManagerInitializer"
        android:value="androidx.startup" />
</provider>
```

---

## Lazy Initialization

```kotlin
// ✅ Initialize on demand — not at startup
class HeavySdkInitializer : Initializer<HeavySdk> {
    override fun create(context: Context): HeavySdk {
        return HeavySdk.initialize(context)
    }
    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// ✅ Disable auto-init in manifest — initialize manually when needed
// In manifest:
// <meta-data android:name="com.example.HeavySdkInitializer" android:value="androidx.startup" />
// Remove this entry to disable auto-init

// Initialize manually when first needed
AppInitializer.getInstance(context)
    .initializeComponent(HeavySdkInitializer::class.java)
```

---

## Disabling Third-Party SDK Auto-Init

```xml
<!-- ✅ Disable WorkManager's own ContentProvider to avoid duplicate init -->
<provider
    android:name="androidx.work.impl.WorkManagerInitializer"
    android:authorities="${applicationId}.workmanager-init"
    android:exported="false"
    tools:node="remove" />

<!-- ✅ Disable Firebase auto-init if using App Startup instead -->
<provider
    android:name="com.google.firebase.provider.FirebaseInitProvider"
    android:authorities="${applicationId}.firebaseinitprovider"
    tools:node="remove" />
```

---

## Measuring Startup Impact

```kotlin
// ✅ Use systrace to see initializer timing
// In Initializer.create():
Trace.beginSection("TimberInitializer")
try {
    Timber.plant(Timber.DebugTree())
} finally {
    Trace.endSection()
}
```

---

## Anti-Patterns

- Doing network calls inside `Initializer.create()` — runs on main thread
- Not declaring dependencies between initializers — ordering not guaranteed
- Registering all SDKs eagerly when most can be lazy — slows startup
- Initializing inside `Application.onCreate()` AND via App Startup — double initialization
- Ignoring `tools:node="remove"` for SDKs with their own ContentProviders — duplicate init

---

## Related Skills
- `startup-optimization` — broader startup performance strategy
- `baseline-profile` — pre-compiling initialization code paths
- `workmanager` — WorkManager initialization via App Startup
- `lifecycle` — deferring init beyond startup using lifecycle observers
