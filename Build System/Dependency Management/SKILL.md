---
name: dependency-management
description: >
  Gradle dependency management — conflict resolution, dependency configurations,
  transitive dependencies, and keeping dependencies up to date.
  Load this skill when resolving version conflicts, understanding dependency
  graphs, or managing third-party library upgrades.
---

# Dependency Management

## Overview
Dependency management covers how Gradle resolves, configures, and maintains third-party libraries. Conflicts arise when multiple libraries require different versions of the same transitive dependency. Understanding configurations (`implementation`, `api`, `compileOnly`) and resolution strategies prevents runtime crashes and build issues.

---

## Core Principles

- Use `implementation` by default — expose via `api` only when the type is part of your public API
- Let Version Catalog be the **single source of truth** for all versions
- Resolve conflicts **explicitly** — never rely on Gradle's default resolution silently winning
- Keep dependencies **up to date** — stale deps accumulate security vulnerabilities
- Audit the **dependency tree** regularly — transitive deps can surprise you

---

## Dependency Configurations

```kotlin
dependencies {
    // ✅ implementation — compile + runtime, NOT exposed to consumers
    implementation(libs.retrofit)

    // ✅ api — compile + runtime, EXPOSED to consumers (module's public API)
    api(libs.kotlinx.coroutines.core)  // use when consumers need this type

    // ✅ compileOnly — compile time only, not in runtime classpath
    compileOnly(libs.javax.annotation)

    // ✅ runtimeOnly — runtime only, not available at compile time
    runtimeOnly(libs.room.runtime)

    // ✅ testImplementation — test compile + runtime only
    testImplementation(libs.junit)

    // ✅ androidTestImplementation — instrumented test only
    androidTestImplementation(libs.espresso.core)

    // ✅ debugImplementation — debug build type only
    debugImplementation(libs.leakcanary)

    // ✅ ksp — annotation processor (KSP)
    ksp(libs.hilt.compiler)
}
```

---

## Viewing the Dependency Tree

```bash
# ✅ Full dependency tree for a configuration
./gradlew :app:dependencies --configuration debugRuntimeClasspath

# ✅ Find who brings a specific dependency
./gradlew :app:dependencyInsight \
    --configuration debugRuntimeClasspath \
    --dependency okhttp

# ✅ Per-module dependency tree
./gradlew :core:network:dependencies
```

---

## Resolving Version Conflicts

```kotlin
// ✅ Force a specific version globally
configurations.all {
    resolutionStrategy {
        force("com.squareup.okhttp3:okhttp:4.12.0")
        force("org.jetbrains.kotlin:kotlin-stdlib:2.0.0")
    }
}

// ✅ Prefer highest version (Gradle default — explicit for clarity)
configurations.all {
    resolutionStrategy.eachDependency {
        if (requested.group == "com.squareup.okhttp3") {
            useVersion("4.12.0")
            because("Align all OkHttp modules to same version")
        }
    }
}

// ✅ Exclude a transitive dependency
implementation(libs.some.library) {
    exclude(group = "com.google.guava", module = "guava")
}

// ✅ Exclude globally
configurations.all {
    exclude(group = "commons-logging", module = "commons-logging")
}
```

---

## Platform BOM (Bill of Materials)

```kotlin
// ✅ BOM aligns all library versions in a group
dependencies {
    // Firebase BOM — manages all Firebase SDK versions
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)   // no version needed
    implementation(libs.firebase.crashlytics) // no version needed

    // Compose BOM
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)           // no version needed
    implementation(libs.compose.material3)    // no version needed
}
```

---

## Dependency Locking

```kotlin
// ✅ Lock dependency versions for reproducible builds
// In settings.gradle.kts or build.gradle.kts
dependencyLocking {
    lockAllConfigurations()
}

// Generate lock files
// ./gradlew dependencies --write-locks

// Verify locked versions
// ./gradlew dependencies --verify-metadata
```

```
# gradle/dependency-locks/*.lockfile — commit these to version control
# They ensure every developer and CI uses identical dependency versions
```

---

## Keeping Dependencies Updated

```bash
# ✅ Use Gradle Versions Plugin
plugins {
    id("com.github.ben-manes.versions") version "0.51.0"
}

# Check for updates
./gradlew dependencyUpdates

# Report format options:
./gradlew dependencyUpdates -Drevision=release
```

```kotlin
// ✅ Configure to only suggest stable releases
tasks.withType<DependencyUpdatesTask> {
    rejectVersionIf {
        isNonStable(candidate.version) && !isNonStable(currentVersion)
    }
}

fun isNonStable(version: String): Boolean {
    val unstableKeywords = listOf("alpha", "beta", "rc", "cr", "m", "preview", "eap")
    return unstableKeywords.any { version.lowercase().contains(it) }
}
```

---

## Module Dependency Graph

```kotlin
// ✅ Enforce no circular dependencies between modules
// In build-logic convention plugin
configurations.all {
    resolutionStrategy.dependencySubstitution {
        // Feature modules must not depend on each other
    }
}

// ✅ Use module dependency graph validation (see module-dependency-graph-validation skill)
```

---

## Common Conflict Scenarios

```kotlin
// Scenario 1: Two libraries require different OkHttp versions
// Library A requires okhttp:3.x, Library B requires okhttp:4.x
// Gradle picks 4.x by default (highest) — usually fine but verify

// ✅ Verify by checking the dependency tree:
// ./gradlew :app:dependencyInsight --dependency okhttp

// Scenario 2: Kotlin stdlib version mismatch
// ✅ Force Kotlin stdlib version to match your Kotlin version
configurations.all {
    resolutionStrategy.eachDependency {
        if (requested.group == "org.jetbrains.kotlin" &&
            requested.name.startsWith("kotlin-")) {
            useVersion(libs.versions.kotlin.get())
            because("Align all Kotlin modules to project Kotlin version")
        }
    }
}

// Scenario 3: Duplicate classes from different artifacts
// Example: guava-jre vs guava-android
implementation(libs.guava) {
    exclude(group = "com.google.guava", module = "guava-jre")
}
```

---

## `api` vs `implementation` in Library Modules

```kotlin
// ✅ Use api() when consumers need the type in their code
// core/network module:
api(libs.retrofit)         // ✅ consumers call Retrofit types in their code
implementation(libs.okhttp) // ✅ consumers don't see OkHttp directly

// ✅ Use implementation() for everything else
// Leaking implementation details via api() increases compilation coupling
// — changes to api() deps trigger recompilation of all consumers
```

---

## Anti-Patterns

- Using `api()` for everything — every change triggers full recompilation
- Ignoring version conflict warnings — silent resolution may pick wrong version
- Not using BOM for library groups (Firebase, Compose) — version mismatches
- Transitive dependency assumption — a dep available via transitive might disappear
- No dependency locking in CI — different devs get different dependency versions
- Outdated dependencies — security vulnerabilities accumulate silently

---

## Related Skills
- `version-catalog` — centralized version definitions
- `gradle` — Gradle build configuration
- `build-cache` — caching with consistent dependencies
- `dependency-compatibility-resolver` — automated conflict resolution
- `module-dependency-graph-validation` — enforcing module boundaries
