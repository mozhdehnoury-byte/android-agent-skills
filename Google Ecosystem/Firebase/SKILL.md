---
name: firebase
description: >
  Firebase setup and core configuration for Android.
  Load this skill when initializing Firebase, configuring google-services.json,
  setting up multiple Firebase environments (dev/prod), or integrating Firebase
  into a multi-module project.
---

# Firebase

## Overview
Firebase is Google's app development platform providing backend services — authentication, messaging, crash reporting, analytics, remote config, and more. Each service is a separate SDK added independently. The core setup is shared across all Firebase services.

---

## Core Principles

- One `google-services.json` per app variant — dev and prod must use separate Firebase projects
- Initialize Firebase **once** in `Application.onCreate()` — never in Activity or Fragment
- Use **separate Firebase projects** for debug and release — never mix environments
- Add only the Firebase SDKs you actually use — each adds APK size and startup overhead
- Never commit `google-services.json` with production credentials to public repos

---

## Setup

```toml
[versions]
firebase-bom = "33.5.1"
google-services = "4.4.2"

[libraries]
firebase-bom = { module = "com.google.firebase:firebase-bom", version.ref = "firebase-bom" }
# Individual SDKs — version managed by BOM
firebase-analytics    = { module = "com.google.firebase:firebase-analytics-ktx" }
firebase-crashlytics  = { module = "com.google.firebase:firebase-crashlytics-ktx" }
firebase-messaging    = { module = "com.google.firebase:firebase-messaging-ktx" }
firebase-auth         = { module = "com.google.firebase:firebase-auth-ktx" }
firebase-config       = { module = "com.google.firebase:firebase-config-ktx" }

[plugins]
google-services  = { id = "com.google.gms.google-services", version.ref = "google-services" }
firebase-crashlytics = { id = "com.google.firebase.crashlytics", version = "3.0.2" }
```

```kotlin
// build.gradle.kts (root)
plugins {
    alias(libs.plugins.google.services) apply false
    alias(libs.plugins.firebase.crashlytics) apply false
}

// build.gradle.kts (app)
plugins {
    alias(libs.plugins.google.services)
    alias(libs.plugins.firebase.crashlytics)
}

dependencies {
    implementation(platform(libs.firebase.bom))  // ✅ BOM manages all versions
    implementation(libs.firebase.analytics)
    implementation(libs.firebase.crashlytics)
    implementation(libs.firebase.messaging)
}
```

---

## Application Setup

```kotlin
// ✅ Firebase auto-initializes via ContentProvider — no manual init needed
// FirebaseApp.initializeApp() is called automatically

@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Firebase is already initialized here
        // Configure per-service settings if needed
    }
}
```

---

## Multiple Environments (Dev / Prod)

```
// ✅ Separate google-services.json per build variant
app/
├── src/
│   ├── debug/
│   │   └── google-services.json    ← dev Firebase project
│   └── release/
│       └── google-services.json    ← prod Firebase project
```

```kotlin
// ✅ Or configure via build flavors
android {
    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
        }
        create("prod") {
            dimension = "environment"
        }
    }
}

// Files:
// app/src/dev/google-services.json
// app/src/prod/google-services.json
```

---

## Manual Firebase App Initialization (Multi-Module / KMP)

```kotlin
// ✅ When automatic initialization isn't sufficient
val options = FirebaseOptions.Builder()
    .setApiKey(BuildConfig.FIREBASE_API_KEY)
    .setApplicationId(BuildConfig.FIREBASE_APP_ID)
    .setProjectId(BuildConfig.FIREBASE_PROJECT_ID)
    .build()

FirebaseApp.initializeApp(context, options, "secondary")
val secondaryApp = Firebase.app("secondary")
val secondaryDb = Firebase.database(secondaryApp)
```

---

## Hilt Integration

```kotlin
// ✅ Provide Firebase services via Hilt
@Module
@InstallIn(SingletonComponent::class)
object FirebaseModule {

    @Provides
    @Singleton
    fun provideFirebaseAnalytics(): FirebaseAnalytics =
        FirebaseAnalytics.getInstance(get())  // or inject context

    @Provides
    @Singleton
    fun provideFirebaseRemoteConfig(): FirebaseRemoteConfig =
        FirebaseRemoteConfig.getInstance().also { config ->
            config.setConfigSettingsAsync(
                remoteConfigSettings {
                    minimumFetchIntervalInSeconds = if (BuildConfig.DEBUG) 0 else 3600
                }
            )
        }
}
```

---

## Disable Firebase in Debug

```kotlin
// ✅ Disable analytics and crashlytics in debug builds
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        if (BuildConfig.DEBUG) {
            FirebaseAnalytics.getInstance(this).setAnalyticsCollectionEnabled(false)
            FirebaseCrashlytics.getInstance().setCrashlyticsCollectionEnabled(false)
        }
    }
}
```

---

## Anti-Patterns

- Single Firebase project for dev and prod — production data polluted with test events
- Committing `google-services.json` with prod keys to public repo — security risk
- Initializing Firebase in Activity — causes multiple initializations
- Adding all Firebase SDKs without using them — increases APK size and startup time
- Not using BOM — version conflicts between Firebase SDKs

---

## Related Skills
- `firebase-messaging` — push notifications with FCM
- `crashlytics` — crash reporting setup
- `analytics` — event tracking
- `remote-config` — feature flags from Firebase
- `firebase-auth` — authentication (not in current skill list — add if needed)
