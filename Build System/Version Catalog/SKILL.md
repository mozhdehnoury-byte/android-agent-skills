---
name: version-catalog
description: >
  Gradle Version Catalog setup and usage for Android projects.
  Load this skill when managing dependency versions centrally,
  adding new dependencies, or organizing libs.versions.toml.
---

# Version Catalog

## Overview
Gradle Version Catalog (`libs.versions.toml`) is the standard way to centralize all dependency versions in one file. It provides type-safe accessors in build scripts, prevents version conflicts, and makes upgrades visible in a single diff.

---

## Core Principles

- **All dependencies** go through Version Catalog — no inline versions anywhere
- Group related libraries under the same version alias when possible
- Use **bundles** for sets of dependencies that always go together
- Keep `libs.versions.toml` the **single source of truth** for all versions
- Separate **plugin versions** from library versions clearly

---

## File Structure

```toml
# gradle/libs.versions.toml

[versions]
# SDK
android-compileSdk = "35"
android-minSdk = "24"
android-targetSdk = "35"

# Core
kotlin = "2.0.0"
ksp = "2.0.0-1.0.21"
agp = "8.7.0"

# AndroidX
androidx-core = "1.13.1"
androidx-lifecycle = "2.8.6"
androidx-navigation = "2.8.3"
androidx-room = "2.6.1"
androidx-datastore = "1.1.1"
androidx-paging = "3.3.2"
androidx-work = "2.9.1"

# Compose
compose-bom = "2024.10.01"
compose-compiler = "1.5.14"

# Network
retrofit = "2.11.0"
okhttp = "4.12.0"
ktor = "2.3.12"

# DI
hilt = "2.51.1"
hilt-navigation = "1.2.0"

# Firebase
firebase-bom = "33.5.1"
google-services = "4.4.2"
firebase-crashlytics-plugin = "3.0.2"

# Serialization
kotlinx-serialization = "1.7.3"
kotlinx-coroutines = "1.9.0"

# Media
media3 = "1.4.0"
camerax = "1.3.4"

# Testing
junit = "4.13.2"
junit5 = "5.10.2"
mockk = "1.13.12"
turbine = "1.1.0"
androidx-test-runner = "1.6.2"
espresso = "3.6.1"

# Quality
detekt = "1.23.7"
ktlint = "12.1.1"


[libraries]
# AndroidX Core
androidx-core-ktx = { module = "androidx.core:core-ktx", version.ref = "androidx-core" }
androidx-appcompat = { module = "androidx.appcompat:appcompat", version = "1.7.0" }
androidx-activity-compose = { module = "androidx.activity:activity-compose", version = "1.9.2" }

# Lifecycle
androidx-lifecycle-viewmodel = { module = "androidx.lifecycle:lifecycle-viewmodel-ktx", version.ref = "androidx-lifecycle" }
androidx-lifecycle-runtime = { module = "androidx.lifecycle:lifecycle-runtime-ktx", version.ref = "androidx-lifecycle" }
androidx-lifecycle-runtime-compose = { module = "androidx.lifecycle:lifecycle-runtime-compose", version.ref = "androidx-lifecycle" }

# Compose
compose-bom = { module = "androidx.compose:compose-bom", version.ref = "compose-bom" }
compose-ui = { module = "androidx.compose.ui:ui" }
compose-ui-tooling = { module = "androidx.compose.ui:ui-tooling" }
compose-ui-tooling-preview = { module = "androidx.compose.ui:ui-tooling-preview" }
compose-foundation = { module = "androidx.compose.foundation:foundation" }
compose-material3 = { module = "androidx.compose.material3:material3" }
compose-material-icons = { module = "androidx.compose.material:material-icons-extended" }
compose-ui-test-manifest = { module = "androidx.compose.ui:ui-test-manifest" }
compose-ui-test-junit4 = { module = "androidx.compose.ui:ui-test-junit4" }

# Navigation
androidx-navigation-compose = { module = "androidx.navigation:navigation-compose", version.ref = "androidx-navigation" }

# Room
androidx-room-runtime = { module = "androidx.room:room-runtime", version.ref = "androidx-room" }
androidx-room-ktx = { module = "androidx.room:room-ktx", version.ref = "androidx-room" }
androidx-room-compiler = { module = "androidx.room:room-compiler", version.ref = "androidx-room" }
androidx-room-paging = { module = "androidx.room:room-paging", version.ref = "androidx-room" }
androidx-room-testing = { module = "androidx.room:room-testing", version.ref = "androidx-room" }

# DataStore
androidx-datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "androidx-datastore" }
androidx-datastore-proto = { module = "androidx.datastore:datastore", version.ref = "androidx-datastore" }

# Paging
androidx-paging-runtime = { module = "androidx.paging:paging-runtime", version.ref = "androidx-paging" }
androidx-paging-compose = { module = "androidx.paging:paging-compose", version.ref = "androidx-paging" }

# WorkManager
androidx-work-runtime = { module = "androidx.work:work-runtime-ktx", version.ref = "androidx-work" }
androidx-work-hilt = { module = "androidx.hilt:hilt-work", version = "1.2.0" }
androidx-work-testing = { module = "androidx.work:work-testing", version.ref = "androidx-work" }

# Network
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit-kotlinx-serialization = { module = "com.squareup.retrofit2:converter-kotlinx-serialization", version.ref = "retrofit" }
okhttp = { module = "com.squareup.okhttp3:okhttp", version.ref = "okhttp" }
okhttp-logging = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }
ktor-client-core = { module = "io.ktor:ktor-client-core", version.ref = "ktor" }
ktor-client-okhttp = { module = "io.ktor:ktor-client-okhttp", version.ref = "ktor" }
ktor-client-content-negotiation = { module = "io.ktor:ktor-client-content-negotiation", version.ref = "ktor" }
ktor-serialization-json = { module = "io.ktor:ktor-serialization-kotlinx-json", version.ref = "ktor" }

# DI — Hilt
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hilt" }
hilt-compiler = { module = "com.google.dagger:hilt-compiler", version.ref = "hilt" }
hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version.ref = "hilt-navigation" }
hilt-testing = { module = "com.google.dagger:hilt-android-testing", version.ref = "hilt" }

# Serialization / Coroutines
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinx-serialization" }
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinx-coroutines" }
kotlinx-coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "kotlinx-coroutines" }
kotlinx-coroutines-test = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-test", version.ref = "kotlinx-coroutines" }

# Firebase
firebase-bom = { module = "com.google.firebase:firebase-bom", version.ref = "firebase-bom" }
firebase-analytics = { module = "com.google.firebase:firebase-analytics-ktx" }
firebase-crashlytics = { module = "com.google.firebase:firebase-crashlytics-ktx" }
firebase-messaging = { module = "com.google.firebase:firebase-messaging-ktx" }
firebase-config = { module = "com.google.firebase:firebase-config-ktx" }
firebase-auth = { module = "com.google.firebase:firebase-auth-ktx" }

# Media
media3-exoplayer = { module = "androidx.media3:media3-exoplayer", version.ref = "media3" }
media3-exoplayer-hls = { module = "androidx.media3:media3-exoplayer-hls", version.ref = "media3" }
media3-ui = { module = "androidx.media3:media3-ui", version.ref = "media3" }
media3-session = { module = "androidx.media3:media3-session", version.ref = "media3" }
camerax-core = { module = "androidx.camera:camera-core", version.ref = "camerax" }
camerax-camera2 = { module = "androidx.camera:camera-camera2", version.ref = "camerax" }
camerax-lifecycle = { module = "androidx.camera:camera-lifecycle", version.ref = "camerax" }
camerax-video = { module = "androidx.camera:camera-video", version.ref = "camerax" }
camerax-view = { module = "androidx.camera:camera-view", version.ref = "camerax" }

# Security
androidx-security-crypto = { module = "androidx.security:security-crypto", version = "1.1.0-alpha06" }

# Testing
junit = { module = "junit:junit", version.ref = "junit" }
mockk = { module = "io.mockk:mockk", version.ref = "mockk" }
mockk-android = { module = "io.mockk:mockk-android", version.ref = "mockk" }
turbine = { module = "app.cash.turbine:turbine", version.ref = "turbine" }
androidx-test-runner = { module = "androidx.test:runner", version.ref = "androidx-test-runner" }
espresso-core = { module = "androidx.test.espresso:espresso-core", version.ref = "espresso" }
androidx-junit = { module = "androidx.test.ext:junit", version = "1.2.1" }

# Quality
detekt-formatting = { module = "io.gitlab.arturbosch.detekt:detekt-formatting", version.ref = "detekt" }


[bundles]
# ✅ Bundles — groups of always-used-together dependencies
compose = [
    "compose-ui",
    "compose-foundation",
    "compose-material3",
    "compose-ui-tooling-preview",
    "compose-material-icons"
]

lifecycle = [
    "androidx-lifecycle-viewmodel",
    "androidx-lifecycle-runtime",
    "androidx-lifecycle-runtime-compose"
]

room = [
    "androidx-room-runtime",
    "androidx-room-ktx"
]

network = [
    "retrofit",
    "retrofit-kotlinx-serialization",
    "okhttp",
    "okhttp-logging"
]

coroutines = [
    "kotlinx-coroutines-core",
    "kotlinx-coroutines-android"
]

testing-unit = [
    "junit",
    "mockk",
    "kotlinx-coroutines-test",
    "turbine"
]

testing-android = [
    "androidx-test-runner",
    "androidx-junit",
    "espresso-core"
]


[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
compose-compiler = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
google-services = { id = "com.google.gms.google-services", version.ref = "google-services" }
firebase-crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "firebase-crashlytics-plugin" }
detekt = { id = "io.gitlab.arturbosch.detekt", version.ref = "detekt" }
ktlint = { id = "org.jlleitschuh.gradle.ktlint", version.ref = "ktlint" }
```

---

## Using in build.gradle.kts

```kotlin
// ✅ Reference via type-safe accessors
dependencies {
    // Single library
    implementation(libs.androidx.core.ktx)
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)

    // Bundle
    implementation(libs.bundles.compose)
    implementation(libs.bundles.room)
    ksp(libs.androidx.room.compiler)

    // BOM
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.analytics)

    // Test
    testImplementation(libs.bundles.testing.unit)
    androidTestImplementation(libs.bundles.testing.android)
}

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}
```

---

## Anti-Patterns

- Inline versions in `build.gradle.kts` — `implementation("com.example:lib:1.0.0")`
- Duplicate version definitions across modules
- Not using bundles for frequently co-used dependencies
- Mixing BOM and explicit versions for same library group
- Not keeping catalog alphabetically sorted — hard to find entries

---

## Related Skills
- `gradle` — Gradle build configuration
- `convention-plugin` — using catalog in shared build logic
- `dependency-management` — resolving conflicts with catalog
- `build-variant` — applying different deps per variant
