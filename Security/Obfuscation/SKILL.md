---
name: obfuscation
description: >
  Code obfuscation for Android apps using R8 and ProGuard.
  Load this skill when configuring obfuscation rules, protecting sensitive
  logic from reverse engineering, keeping required classes unobfuscated,
  or troubleshooting crashes caused by obfuscation.
---

# Obfuscation

## Overview
Obfuscation renames classes, methods, and fields to meaningless names, making reverse-engineered code harder to understand. In Android, R8 (the successor to ProGuard) handles shrinking, obfuscation, and optimization in a single pass. Obfuscation is enabled by default in release builds.

---

## Core Principles

- **Obfuscation ≠ security** — it raises the bar, not a complete defense
- R8 is the default since Android Gradle Plugin 3.4 — prefer R8 over ProGuard
- **Keep rules are critical** — wrong keeps = broken app; missing keeps = crashes
- Always **test release builds** — obfuscation bugs only appear in release
- **Mapping file** must be saved — needed to decode crash stack traces

---

## Enable in build.gradle.kts

```kotlin
// ✅ Release build config
android {
    buildTypes {
        release {
            isMinifyEnabled = true        // enables R8 obfuscation + shrinking
            isShrinkResources = true      // removes unused resources
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }

        // ✅ Optional: test obfuscation in debug
        debug {
            isMinifyEnabled = false
        }
    }
}
```

---

## proguard-rules.pro

```proguard
# ============================================================
# Data classes used with serialization (Gson, kotlinx.serialization, Moshi)
# ============================================================
-keepclassmembers class com.example.app.data.remote.dto.** {
    <fields>;
}
-keep class com.example.app.data.remote.dto.** { *; }

# ============================================================
# Room entities and DAOs
# ============================================================
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class *
-keepclassmembers @androidx.room.Entity class * { <fields>; }

# ============================================================
# Retrofit interfaces
# ============================================================
-keep interface com.example.app.data.remote.api.** { *; }
-keepclassmembernames interface * {
    @retrofit2.http.* <methods>;
}

# ============================================================
# Hilt — generated components must not be obfuscated
# ============================================================
-keep class dagger.hilt.** { *; }
-keep class javax.inject.** { *; }
-keep @dagger.hilt.android.HiltAndroidApp class *
-keep @dagger.hilt.InstallIn class *
-keep @dagger.hilt.android.AndroidEntryPoint class *

# ============================================================
# Parcelable
# ============================================================
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# ============================================================
# Enums
# ============================================================
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# ============================================================
# Serialization — kotlinx.serialization
# ============================================================
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keepclassmembers class kotlinx.serialization.json.** {
    *** Companion;
}
-keepclasseswithmembers class **$$serializer {
    static **$$serializer INSTANCE;
}

# ============================================================
# Crash reporting — keep line numbers for stack traces
# ============================================================
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# ============================================================
# Firebase
# ============================================================
-keep class com.google.firebase.** { *; }
-keep class com.google.android.gms.** { *; }

# ============================================================
# OkHttp / Retrofit
# ============================================================
-dontwarn okhttp3.**
-dontwarn retrofit2.**
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }

# ============================================================
# WebView JS interface
# ============================================================
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}
```

---

## Saving the Mapping File

```kotlin
// ✅ Store mapping file for crash deobfuscation
// The mapping.txt is generated at:
// app/build/outputs/mapping/release/mapping.txt

// ✅ Upload to Firebase Crashlytics automatically
android {
    buildTypes {
        release {
            // Crashlytics uploads mapping.txt automatically when minifyEnabled = true
        }
    }
}

// ✅ Or upload to Play Console manually
// Google Play Console > App Bundle Explorer > Download mapping.txt
```

---

## Decoding Obfuscated Stack Traces

```bash
# ✅ Use retrace to decode a stack trace
# Located at: $ANDROID_SDK/tools/proguard/bin/retrace.sh

retrace.sh mapping.txt obfuscated_stacktrace.txt

# ✅ Or use bundletool / R8's retrace
java -jar r8.jar retrace --map mapping.txt stacktrace.txt
```

---

## Testing Obfuscation

```kotlin
// ✅ Create a release variant that's debuggable for testing obfuscation
android {
    buildTypes {
        create("releaseDebug") {
            initWith(getByName("release"))
            isDebuggable = true
            signingConfig = signingConfigs.getByName("debug")
            applicationIdSuffix = ".releasedebug"
        }
    }
}
```

---

## Common Crash Causes After Obfuscation

| Crash | Cause | Fix |
|---|---|---|
| `ClassNotFoundException` | Class renamed by R8 | Add `-keep class com.example.MyClass` |
| `NoSuchMethodException` | Method renamed | Add `-keepclassmembers` rule |
| Gson deserialization returns null | DTO fields renamed | Keep DTO fields |
| Room `IllegalStateException` | Entity/DAO renamed | Keep Room annotated classes |
| Hilt `ComponentNotFound` | Hilt generated class renamed | Keep Hilt annotations |
| `NullPointerException` in Retrofit | Interface methods renamed | Keep Retrofit interfaces |

---

## Anti-Patterns

- `isMinifyEnabled = false` in release builds — ships unobfuscated code
- `-keep class com.example.** { *; }` — keeps everything, defeats obfuscation
- Discarding the mapping file — crash stack traces become unreadable
- Never testing release builds — obfuscation breaks discovered only in production
- Obfuscation as the only security measure — combine with encryption, root detection, pinning

---

## Related Skills
- `r8` — R8-specific optimization and shrinking beyond obfuscation
- `proguard` — legacy ProGuard rules reference
- `reverse-engineering-resistance` — broader anti-tampering strategy
- `gradle` — build configuration for release variants
- `build-variant` — managing debug/release/staging variants
