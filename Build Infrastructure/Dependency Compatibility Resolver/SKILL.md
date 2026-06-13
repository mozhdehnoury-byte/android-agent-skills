---
name: dependency-compatibility-resolver
description: >
  Detecting and resolving dependency version conflicts in Android/Gradle projects.
  Load this skill when a dependency causes a version conflict, when builds fail
  due to duplicate classes, or when upgrading a library causes transitive
  dependency issues.
---

# Dependency Compatibility Resolver

## Overview
Dependency conflicts occur when multiple libraries require different versions of the same transitive dependency. Gradle resolves these automatically using its "highest version wins" strategy — but this can silently break things. This skill provides systematic techniques to detect, diagnose, and resolve conflicts correctly.

---

## Core Principles

- **Understand the conflict first** — don't just force a version blindly
- "Highest version wins" is Gradle's default — it usually works but not always
- Conflicts between **major versions** are often breaking — require careful testing
- Use **BOM** to align library groups — Firebase, Compose, OkHttp suites
- Document **why** a version was forced — future maintainers need context

---

## Detecting Conflicts

```bash
# ✅ View full dependency tree
./gradlew :app:dependencies --configuration debugRuntimeClasspath

# ✅ Find all versions of a specific dependency
./gradlew :app:dependencyInsight \
    --configuration debugRuntimeClasspath \
    --dependency okhttp

# ✅ Find duplicate classes (build error)
# Error: Duplicate class kotlin.collections.jdk8.CollectionsJDK8Kt found in modules
# → Two libraries bring different versions of kotlin-stdlib

# ✅ Check for version conflicts summary
./gradlew :app:dependencies --configuration debugRuntimeClasspath 2>&1 | grep " -> "
# Lines with " -> " show where Gradle picked a different version than requested
```

---

## Reading the Dependency Tree

```
# ✅ How to read dependency tree output
+--- com.squareup.retrofit2:retrofit:2.11.0
|    +--- com.squareup.okhttp3:okhttp:4.12.0 (*)     ← (*) = already shown
|    \--- com.squareup.okio:okio:3.6.0
\--- com.squareup.okhttp3:logging-interceptor:4.12.0
     \--- com.squareup.okhttp3:okhttp:4.12.0 (*)

# ✅ Version conflict shown as:
+--- com.example:library-a:1.0.0
|    \--- com.squareup.okhttp3:okhttp:3.14.9 -> 4.12.0
#                                           ^^^^^^^^^^
#                                           Gradle upgraded from 3.x to 4.x
```

---

## Resolution Strategies

### Strategy 1: Force a Specific Version

```kotlin
// ✅ Use when you need to pin a transitive dependency
configurations.all {
    resolutionStrategy {
        force("com.squareup.okhttp3:okhttp:4.12.0")
        force("org.jetbrains.kotlin:kotlin-stdlib:2.0.0")
    }
}

// ✅ Document the reason
configurations.all {
    resolutionStrategy.eachDependency {
        if (requested.group == "com.squareup.okhttp3") {
            useVersion("4.12.0")
            because("Library X requires 3.x but we need 4.x features; verified compatible")
        }
    }
}
```

### Strategy 2: Exclude Transitive Dependency

```kotlin
// ✅ When a library brings an unwanted transitive dep
implementation(libs.some.library) {
    exclude(group = "com.google.guava", module = "guava-jre")
    // guava-jre conflicts with guava-android — exclude jre version
}

// ✅ Exclude globally across all configurations
configurations.all {
    exclude(group = "commons-logging", module = "commons-logging")
    // replaced by SLF4J — exclude old binding
}
```

### Strategy 3: Use BOM for Aligned Versions

```kotlin
// ✅ BOM ensures all modules in a suite use the same version
dependencies {
    // OkHttp BOM
    implementation(platform("com.squareup.okhttp3:okhttp-bom:4.12.0"))
    implementation("com.squareup.okhttp3:okhttp")
    implementation("com.squareup.okhttp3:logging-interceptor")
    // Both use 4.12.0 — no conflict possible

    // Firebase BOM
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)
    implementation(libs.firebase.crashlytics)
}
```

### Strategy 4: Substitute a Dependency

```kotlin
// ✅ Replace one dependency with another (e.g., different artifact for Android)
configurations.all {
    resolutionStrategy.dependencySubstitution {
        // Replace guava-jre with guava-android
        substitute(module("com.google.guava:guava-jre"))
            .using(module("com.google.guava:guava:32.1.3-android"))
            .because("Android requires -android variant, not -jre")
    }
}
```

---

## Common Conflict Scenarios

### Kotlin Stdlib Conflict

```kotlin
// Problem: Multiple kotlin-stdlib versions
// Solution: Force to match project Kotlin version
configurations.all {
    resolutionStrategy.eachDependency {
        if (requested.group == "org.jetbrains.kotlin" &&
            requested.name.startsWith("kotlin-")) {
            useVersion(libs.versions.kotlin.get())
            because("Align all Kotlin modules to project Kotlin version")
        }
    }
}
```

### OkHttp 3 vs 4 Conflict

```kotlin
// Problem: Library A needs OkHttp 3.x, app uses 4.x
// Solution: Force 4.x — OkHttp 4 is mostly backward compatible
configurations.all {
    resolutionStrategy.force("com.squareup.okhttp3:okhttp:4.12.0")
}
// ✅ Test the library that needed 3.x still works
```

### Duplicate Class Error

```kotlin
// Error: Duplicate class kotlin.collections.jdk8.CollectionsJDK8Kt
// Cause: kotlin-stdlib-jdk8 merged into kotlin-stdlib in Kotlin 1.8+
// Solution: Exclude the separate artifact

configurations.all {
    resolutionStrategy.eachDependency {
        if (requested.group == "org.jetbrains.kotlin" &&
            (requested.name == "kotlin-stdlib-jdk7" ||
             requested.name == "kotlin-stdlib-jdk8")) {
            useTarget("org.jetbrains.kotlin:kotlin-stdlib:${requested.version}")
            because("kotlin-stdlib-jdk7/8 merged into kotlin-stdlib since 1.8")
        }
    }
}
```

### Guava JRE vs Android

```kotlin
// Problem: Some libraries bring guava-jre which has Java 8+ APIs not on Android
configurations.all {
    resolutionStrategy.dependencySubstitution {
        substitute(module("com.google.guava:guava"))
            .using(module("com.google.guava:guava:32.1.3-android"))
            .because("Android requires -android variant")
    }
}
```

---

## Verification After Resolution

```bash
# ✅ Verify the resolved version is what you expect
./gradlew :app:dependencyInsight \
    --dependency okhttp \
    --configuration debugRuntimeClasspath

# ✅ Run tests to verify compatibility
./gradlew test
./gradlew connectedAndroidTest

# ✅ Check for duplicate classes
./gradlew assembleDebug 2>&1 | grep "Duplicate class"
```

---

## Documenting Resolutions

```kotlin
// ✅ Always explain why a version was forced
configurations.all {
    resolutionStrategy.eachDependency {
        when {
            requested.group == "com.squareup.okhttp3" -> {
                useVersion("4.12.0")
                because(
                    "Library X still requests OkHttp 3.14 but is compatible with 4.x. " +
                    "Verified with X's test suite on 2024-11-15. " +
                    "Remove this override when X releases 2.0 with native OkHttp 4 support."
                )
            }
        }
    }
}
```

---

## Anti-Patterns

- Blindly forcing a version without testing — can break the library that needed the old version
- Not documenting resolution reasons — future maintainers don't know why it's forced
- Excluding a dep without verifying the library works without it — runtime crash
- Ignoring `-> x.y.z` in dependency tree — silent upgrades can break things
- Forcing an older version when Gradle upgraded for good reason — misses security fixes

---

## Related Skills
- `dependency-management` — dependency configurations and upgrades
- `version-catalog` — centralized version management
- `gradle` — Gradle build configuration
- `build-cache` — cache invalidation after dependency changes
