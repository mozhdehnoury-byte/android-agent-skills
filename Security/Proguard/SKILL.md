---
name: proguard
description: >
  ProGuard rules reference for Android apps.
  Load this skill when writing or debugging -keep rules, understanding
  ProGuard rule syntax, maintaining proguard-rules.pro, or resolving
  runtime crashes caused by missing keep rules.
---

# ProGuard

## Overview
ProGuard is the rule language used by both the legacy ProGuard tool and its successor R8. In modern Android projects, you write ProGuard-syntax rules in `proguard-rules.pro`, and R8 consumes them. Understanding the rule syntax is essential for preventing runtime crashes in release builds.

---

## Core Principles

- ProGuard rules **whitelist** — everything is removed unless kept
- Rules are **additive** — you cannot un-keep something kept by another rule
- Prefer **specific rules** over broad ones — broad rules defeat shrinking
- Rules affect **shrinking, obfuscation, and optimization** independently
- Test on a real device with a release/minified build before shipping

---

## Rule Syntax Reference

```proguard
# Keep a class and all its members
-keep class com.example.MyClass { *; }

# Keep a class name only (members may still be removed/renamed)
-keep class com.example.MyClass

# Keep members but allow class renaming
-keepclassmembers class com.example.MyClass {
    public void myMethod();
    private String myField;
}

# Keep class if it has certain members (conditional keep)
-keepclasseswithmembers class * {
    public <init>(android.content.Context);
}

# Keep class name if it has certain members (class name kept, members may be renamed)
-keepclasseswithmembernames class * {
    native <methods>;
}

# Keep attributes (metadata)
-keepattributes *Annotation*
-keepattributes Signature
-keepattributes SourceFile,LineNumberTable
-keepattributes Exceptions

# Suppress warnings (use sparingly)
-dontwarn com.example.third.party.**

# Suppress notes
-dontnote com.example.third.party.**

# Rename source file attribute (for stack trace clarity)
-renamesourcefileattribute SourceFile
```

---

## Wildcards

```proguard
# * — matches any single package component or partial name (not dots)
-keep class com.example.*.Model { *; }    # matches com.example.user.Model
                                           # NOT com.example.user.detail.Model

# ** — matches any number of package components (including dots)
-keep class com.example.**.Model { *; }   # matches both

# *** — matches any type (primitive or object)
# % — matches any primitive type
# <fields> — all fields
# <methods> — all methods
# <init> — constructors
```

---

## Common Rules by Library

```proguard
# ── Retrofit ──────────────────────────────────────────────────────────────
-keepattributes Signature
-keepattributes Exceptions
-keep interface * {
    @retrofit2.http.* <methods>;
}
-dontwarn retrofit2.**

# ── OkHttp ────────────────────────────────────────────────────────────────
-dontwarn okhttp3.**
-dontwarn okio.**
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }

# ── Gson ──────────────────────────────────────────────────────────────────
-keepclassmembers class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
-keep class com.example.app.data.remote.dto.** { *; }

# ── kotlinx.serialization ─────────────────────────────────────────────────
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt
-keepclassmembers class kotlinx.serialization.json.** {
    *** Companion;
}

# ── Room ──────────────────────────────────────────────────────────────────
-keep class * extends androidx.room.RoomDatabase
-keep @androidx.room.Entity class * { *; }
-keep @androidx.room.Dao interface * { *; }

# ── Hilt / Dagger ─────────────────────────────────────────────────────────
-keep class dagger.** { *; }
-keep class javax.inject.** { *; }
-keep @dagger.hilt.android.HiltAndroidApp class *
-keep @dagger.hilt.android.AndroidEntryPoint class *
-keep @dagger.hilt.InstallIn class *

# ── Parcelable ────────────────────────────────────────────────────────────
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# ── Enum ──────────────────────────────────────────────────────────────────
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# ── Firebase ──────────────────────────────────────────────────────────────
-keep class com.google.firebase.** { *; }
-keep class com.google.android.gms.** { *; }
-dontwarn com.google.firebase.**

# ── Crashlytics ───────────────────────────────────────────────────────────
-keepattributes SourceFile,LineNumberTable
-keep public class * extends java.lang.Exception
```

---

## Diagnosing Missing Keep Rules

```bash
# Run release build and check logcat for:
# E/AndroidRuntime: java.lang.ClassNotFoundException: com.example.MyClass
# → Add: -keep class com.example.MyClass { *; }

# E/AndroidRuntime: java.lang.NoSuchMethodException: myMethod
# → Add: -keepclassmembers class com.example.MyClass { void myMethod(); }

# Check usage.txt to see what was removed:
# app/build/outputs/mapping/release/usage.txt
```

---

## Rule Debugging

```proguard
# ✅ Print which rules were applied (outputs to build logs)
-printconfiguration build/outputs/mapping/release/configuration.txt

# ✅ Print which classes were kept
-printseeds build/outputs/mapping/release/seeds.txt

# ✅ Print which code was removed
-printusage build/outputs/mapping/release/usage.txt

# ✅ Print the mapping
-printmapping build/outputs/mapping/release/mapping.txt
```

---

## Anti-Patterns

- `-keep class com.example.** { *; }` — keeps all code, no shrinking benefit
- Writing rules without testing the release build — issues only surface at runtime
- `-dontwarn **` — suppresses all warnings including real problems
- Forgetting `-keepattributes Signature` — breaks Retrofit generic type resolution
- Not keeping DTO fields with Gson — deserialized objects have null fields in release

---

## Related Skills
- `r8` — R8 configuration and full mode
- `obfuscation` — obfuscation strategy and when to apply it
- `build-variant` — applying different rules per build type
- `gradle` — build configuration
