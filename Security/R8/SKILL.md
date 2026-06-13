---
name: r8
description: >
  R8 compiler for Android app shrinking, obfuscation, and optimization.
  Load this skill when configuring R8 rules, understanding how R8 differs
  from ProGuard, enabling full mode, troubleshooting R8-related build or
  runtime issues, or optimizing APK/AAB size with R8.
---

# R8

## Overview
R8 is Android's default code shrinker, obfuscator, and optimizer — replacing ProGuard since AGP 3.4. R8 runs as part of the build pipeline and performs three tasks in one pass: **shrinking** (removes unused code), **obfuscation** (renames identifiers), and **optimization** (inlines, removes dead branches, optimizes bytecode).

---

## Core Principles

- R8 is enabled via `isMinifyEnabled = true` — not a separate tool invocation
- R8 Full Mode (`-fullmode`) is more aggressive — requires more explicit keep rules
- R8 consumes the same `-keep` rules as ProGuard — existing rules work
- R8 produces a `mapping.txt` — required for deobfuscating crash stack traces
- Test release builds — R8 transformations can break reflection-heavy code

---

## Build Configuration

```kotlin
// ✅ Standard R8 setup
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true    // removes unused resources (requires minify)
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),  // base rules
                "proguard-rules.pro"    // app-specific rules
            )
        }
    }

    // ✅ Enable R8 full mode for maximum shrinking
    // Add to gradle.properties:
    // android.enableR8.fullMode=true
}
```

---

## R8 vs ProGuard

| Feature | ProGuard | R8 |
|---|---|---|
| Shrinking | ✅ | ✅ |
| Obfuscation | ✅ | ✅ |
| Optimization | Basic | Advanced (inlining, constant propagation) |
| Build speed | Slower | Faster (single pass) |
| Kotlin support | Limited | Full |
| Full mode | ❌ | ✅ (opt-in) |
| Default since | AGP < 3.4 | AGP 3.4+ |

---

## R8 Full Mode

```properties
# gradle.properties — enables more aggressive shrinking
android.enableR8.fullMode=true
```

```proguard
# Full mode removes default constructors — keep them explicitly if needed
-keepclassmembers class com.example.app.data.remote.dto.** {
    <init>();   # keep no-arg constructor for deserialization
    <fields>;
}

# Full mode is stricter about interface default methods
-keep interface com.example.app.domain.repository.** { *; }
```

---

## Essential Keep Rules

```proguard
# ============================================================
# Reflection — anything accessed via reflection must be kept
# ============================================================
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# ============================================================
# kotlinx.serialization
# ============================================================
-keepattributes *Annotation*, InnerClasses
-keepclasseswithmembers class * {
    @kotlinx.serialization.Serializable *;
}

# ============================================================
# Preserve line numbers for crash reports
# ============================================================
-keepattributes SourceFile,LineNumberTable
-renamesourcefileattribute SourceFile

# ============================================================
# Keep generic signatures (needed by Retrofit)
# ============================================================
-keepattributes Signature

# ============================================================
# Native methods
# ============================================================
-keepclasseswithmembernames class * {
    native <methods>;
}

# ============================================================
# Custom views (used in XML layouts)
# ============================================================
-keep public class * extends android.view.View {
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

# ============================================================
# Suppress warnings for known-safe libraries
# ============================================================
-dontwarn kotlin.reflect.jvm.internal.**
-dontwarn org.slf4j.**
```

---

## R8 Optimization Examples

```kotlin
// R8 inlines small functions — no performance penalty for utility wrappers
// Before optimization:
fun isValidEmail(email: String) = email.contains("@")

// R8 may inline this at call sites:
if (isValidEmail(input)) { ... }
// →  if (input.contains("@")) { ... }

// R8 removes unreachable code:
if (BuildConfig.DEBUG) {
    Log.d(TAG, "Debug only")   // removed entirely in release build
}
```

---

## Checking What R8 Removed

```bash
# ✅ Check which classes/methods were removed
# build/outputs/mapping/release/usage.txt — lists removed code

# ✅ Check final APK contents
# Android Studio → Build → Analyze APK

# ✅ R8 produces these files in build/outputs/mapping/release/:
# mapping.txt     — obfuscated → original name mapping
# seeds.txt       — classes/members that matched -keep rules
# usage.txt       — code removed by shrinking
# configuration.txt — merged ProGuard rules applied
```

---

## Library AAR Keep Rules

```proguard
# ✅ If publishing an AAR library, define consumer rules
# In library module: src/main/proguard-consumer-rules.pro
# These are automatically applied to apps that use the library

-keep public class com.example.mylibrary.PublicApi { *; }
-keep public interface com.example.mylibrary.Callback { *; }
```

```kotlin
// ✅ Reference consumer rules in library build.gradle.kts
android {
    defaultConfig {
        consumerProguardFiles("proguard-consumer-rules.pro")
    }
}
```

---

## Troubleshooting Checklist

```
R8 build succeeds but app crashes at runtime:
  1. Check logcat for ClassNotFoundException, NoSuchMethodException
  2. Add -keep rule for the missing class/method
  3. Rebuild and test again

R8 build fails:
  1. Check build output for "R8: error" messages
  2. Look for conflicting keep rules
  3. Try disabling full mode temporarily

APK too large after R8:
  1. Enable isShrinkResources = true
  2. Check if -keep rules are too broad
  3. Use Android Studio APK Analyzer to identify large classes
```

---

## Anti-Patterns

- `isMinifyEnabled = false` in release — ships unoptimized, unobfuscated code
- Broad `-keep com.example.app.** { *; }` rules — keeps entire package, defeats shrinking
- Not saving `mapping.txt` from each release — crashes become undecodable
- Only testing debug builds — R8 transformations only apply in release
- Using ProGuard when R8 is available — R8 is faster and more capable

---

## Related Skills
- `obfuscation` — obfuscation rules and strategy
- `proguard` — ProGuard rules reference for older setups
- `build-variant` — managing release/debug build types
- `gradle` — build configuration
- `reverse-engineering-resistance` — combining R8 with other protections
