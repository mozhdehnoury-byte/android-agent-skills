---
name: gradle
description: >
  Gradle build system setup and configuration for Android projects.
  Load this skill when configuring build.gradle.kts files, setting up
  Android build config, managing build types, or understanding Gradle
  fundamentals in an Android context.
---

# Gradle

## Overview
Gradle is the build system for Android. It compiles code, manages dependencies, runs tests, and packages the APK/AAB. Android projects use the Kotlin DSL (`build.gradle.kts`) — preferred over Groovy DSL. Understanding Gradle's task graph and configuration phase is essential for maintaining fast, reliable builds.

---

## Core Principles

- Use **Kotlin DSL** (`build.gradle.kts`) — not Groovy (`build.gradle`)
- Use **Version Catalog** (`libs.versions.toml`) for all dependency versions
- Use **Convention Plugins** to share build logic across modules
- Never hardcode versions inline — always reference from Version Catalog
- Keep `build.gradle.kts` files **short** — logic belongs in convention plugins

---

## Project Structure

```
project/
├── build.gradle.kts          ← root build file (minimal)
├── settings.gradle.kts       ← module declarations
├── gradle/
│   ├── libs.versions.toml    ← version catalog
│   └── wrapper/
│       └── gradle-wrapper.properties
├── build-logic/              ← convention plugins
│   └── convention/
│       └── src/main/kotlin/
│           ├── AndroidApplicationConventionPlugin.kt
│           └── AndroidLibraryConventionPlugin.kt
├── app/
│   └── build.gradle.kts
└── feature/
    └── build.gradle.kts
```

---

## settings.gradle.kts

```kotlin
pluginManagement {
    includeBuild("build-logic")
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}

rootProject.name = "MyApp"

include(":app")
include(":feature:home")
include(":feature:auth")
include(":core:network")
include(":core:database")
```

---

## Root build.gradle.kts

```kotlin
// ✅ Root build file — minimal, only plugin declarations
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.serialization) apply false
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.ksp) apply false
    alias(libs.plugins.google.services) apply false
    alias(libs.plugins.firebase.crashlytics) apply false
}
```

---

## App Module build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.myapp.android.application)  // convention plugin
    alias(libs.plugins.myapp.android.hilt)
    alias(libs.plugins.google.services)
}

android {
    namespace = "com.example.app"

    defaultConfig {
        applicationId = "com.example.app"
        versionCode = 1
        versionName = "1.0.0"

        // ✅ Build config fields for environment-specific values
        buildConfigField("String", "API_BASE_URL", "\"https://api.example.com/\"")
    }

    buildTypes {
        debug {
            isDebuggable = true
            applicationIdSuffix = ".debug"
            buildConfigField("String", "API_BASE_URL", "\"https://dev.api.example.com/\"")
        }
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            signingConfig = signingConfigs.getByName("release")
        }
    }

    buildFeatures {
        buildConfig = true
        compose = true
    }
}

dependencies {
    implementation(projects.core.network)
    implementation(projects.core.database)
    implementation(projects.feature.home)

    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)
}
```

---

## Library Module build.gradle.kts

```kotlin
plugins {
    alias(libs.plugins.myapp.android.library)  // convention plugin
    alias(libs.plugins.myapp.android.hilt)
}

android {
    namespace = "com.example.core.network"
}

dependencies {
    implementation(libs.retrofit)
    implementation(libs.okhttp)
    implementation(libs.kotlinx.serialization.json)
    api(libs.retrofit.kotlinx.serialization)  // ✅ api() exposes to consumers
}
```

---

## Build Config Access

```kotlin
// ✅ Access BuildConfig fields in code
val baseUrl = BuildConfig.API_BASE_URL
val isDebug = BuildConfig.DEBUG
val versionName = BuildConfig.VERSION_NAME

// ✅ Enable BuildConfig generation
android {
    buildFeatures {
        buildConfig = true
    }
}
```

---

## Signing Configuration

```kotlin
// ✅ Read signing config from environment — never hardcode in build file
android {
    signingConfigs {
        create("release") {
            storeFile = file(System.getenv("KEYSTORE_PATH") ?: "keystore.jks")
            storePassword = System.getenv("KEYSTORE_PASSWORD")
            keyAlias = System.getenv("KEY_ALIAS")
            keyPassword = System.getenv("KEY_PASSWORD")
        }
    }
}
```

---

## Useful Gradle Tasks

```bash
# Build
./gradlew assembleDebug
./gradlew assembleRelease
./gradlew bundleRelease          # AAB for Play Store

# Test
./gradlew test                   # unit tests
./gradlew connectedAndroidTest   # instrumented tests

# Analysis
./gradlew lint
./gradlew detekt
./gradlew dependencies           # dependency tree
./gradlew :app:dependencies --configuration debugRuntimeClasspath

# Build performance
./gradlew assembleDebug --profile   # generates HTML report
./gradlew assembleDebug --scan      # Gradle build scan

# Clean
./gradlew clean
```

---

## Gradle Properties

```properties
# gradle.properties

# ✅ Performance
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true

# ✅ Android
android.useAndroidX=true
android.nonTransitiveRClass=true
android.enableR8.fullMode=true

# ✅ Kotlin
kotlin.code.style=official
```

---

## Anti-Patterns

- Groovy DSL (`build.gradle`) in new projects — use Kotlin DSL
- Hardcoded versions inline — use Version Catalog
- Business logic in `build.gradle.kts` — use convention plugins
- `implementation` everywhere — use `api()` when the type is part of public API
- Not enabling `nonTransitiveRClass` — slows builds in multi-module projects
- Committing signing credentials in build files — use environment variables

---

## Related Skills
- `version-catalog` — managing dependency versions
- `convention-plugin` — shared build logic
- `build-variant` — build types and flavors
- `dependency-management` — resolving dependency conflicts
- `build-cache` — caching build outputs
