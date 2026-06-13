---
name: convention-plugin
description: >
  Gradle convention plugins for sharing build logic across modules.
  Load this skill when creating reusable build configuration,
  eliminating duplication across module build files, or setting up
  a build-logic module.
---

# Convention Plugin

## Overview
Convention plugins are Gradle plugins written in `build-logic/` that encapsulate shared build configuration. Instead of duplicating Android, Kotlin, and Hilt setup in every module's `build.gradle.kts`, each module applies a single convention plugin that contains all the shared logic.

---

## Core Principles

- Convention plugins live in `build-logic/` — a separate included build
- Each plugin covers **one concern** — Android app, Android library, Compose, Hilt, etc.
- Modules apply **multiple small plugins** rather than one large plugin
- Convention plugins use the **Version Catalog** for all dependency/version references
- Changes to a convention plugin propagate to **all modules** that apply it

---

## Project Structure

```
build-logic/
├── settings.gradle.kts
└── convention/
    ├── build.gradle.kts
    └── src/main/kotlin/
        ├── AndroidApplicationConventionPlugin.kt
        ├── AndroidLibraryConventionPlugin.kt
        ├── AndroidComposeConventionPlugin.kt
        ├── AndroidHiltConventionPlugin.kt
        ├── AndroidRoomConventionPlugin.kt
        └── KotlinSerializationConventionPlugin.kt
```

---

## build-logic Setup

```kotlin
// build-logic/settings.gradle.kts
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
    versionCatalogs {
        create("libs") {
            from(files("../gradle/libs.versions.toml"))
        }
    }
}
rootProject.name = "build-logic"
include(":convention")
```

```kotlin
// build-logic/convention/build.gradle.kts
plugins {
    `kotlin-dsl`
}

dependencies {
    compileOnly(libs.android.gradlePlugin)
    compileOnly(libs.kotlin.gradlePlugin)
    compileOnly(libs.ksp.gradlePlugin)
}
```

```toml
# Add to libs.versions.toml for use in build-logic
[libraries]
android-gradlePlugin = { module = "com.android.tools.build:gradle", version.ref = "agp" }
kotlin-gradlePlugin  = { module = "org.jetbrains.kotlin:kotlin-gradle-plugin", version.ref = "kotlin" }
ksp-gradlePlugin     = { module = "com.google.devtools.ksp:com.google.devtools.ksp.gradle.plugin", version.ref = "ksp" }
```

---

## Android Application Plugin

```kotlin
// AndroidApplicationConventionPlugin.kt
class AndroidApplicationConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.application")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<ApplicationExtension> {
                compileSdk = libs.findVersion("android-compileSdk").get().toString().toInt()

                defaultConfig {
                    minSdk = libs.findVersion("android-minSdk").get().toString().toInt()
                    targetSdk = libs.findVersion("android-targetSdk").get().toString().toInt()
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                }

                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }

                kotlinOptions { jvmTarget = "17" }
            }
        }
    }
}

private val Project.libs
    get(): VersionCatalog = extensions.getByType<VersionCatalogsExtension>().named("libs")
```

---

## Android Library Plugin

```kotlin
// AndroidLibraryConventionPlugin.kt
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.android.library")
                apply("org.jetbrains.kotlin.android")
            }

            extensions.configure<LibraryExtension> {
                compileSdk = libs.findVersion("android-compileSdk").get().toString().toInt()

                defaultConfig {
                    minSdk = libs.findVersion("android-minSdk").get().toString().toInt()
                    testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
                    consumerProguardFiles("consumer-rules.pro")
                }

                compileOptions {
                    sourceCompatibility = JavaVersion.VERSION_17
                    targetCompatibility = JavaVersion.VERSION_17
                }

                kotlinOptions { jvmTarget = "17" }
            }
        }
    }
}
```

---

## Compose Plugin

```kotlin
// AndroidComposeConventionPlugin.kt
class AndroidComposeConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.jetbrains.kotlin.plugin.compose")

            val extension = extensions.findByType<CommonExtension<*, *, *, *, *, *>>()
                ?: error("Android plugin must be applied first")

            extension.apply {
                buildFeatures { compose = true }
            }

            dependencies {
                val bom = libs.findLibrary("compose-bom").get()
                add("implementation", platform(bom))
                add("implementation", libs.findLibrary("compose-ui").get())
                add("implementation", libs.findLibrary("compose-material3").get())
                add("implementation", libs.findLibrary("compose-ui-tooling-preview").get())
                add("debugImplementation", libs.findLibrary("compose-ui-tooling").get())
            }
        }
    }
}
```

---

## Hilt Plugin

```kotlin
// AndroidHiltConventionPlugin.kt
class AndroidHiltConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("com.google.dagger.hilt.android")
                apply("com.google.devtools.ksp")
            }

            dependencies {
                add("implementation", libs.findLibrary("hilt-android").get())
                add("ksp", libs.findLibrary("hilt-compiler").get())
            }
        }
    }
}
```

---

## Registering Plugins

```kotlin
// build-logic/convention/build.gradle.kts
gradlePlugin {
    plugins {
        register("androidApplication") {
            id = "myapp.android.application"
            implementationClass = "AndroidApplicationConventionPlugin"
        }
        register("androidLibrary") {
            id = "myapp.android.library"
            implementationClass = "AndroidLibraryConventionPlugin"
        }
        register("androidCompose") {
            id = "myapp.android.compose"
            implementationClass = "AndroidComposeConventionPlugin"
        }
        register("androidHilt") {
            id = "myapp.android.hilt"
            implementationClass = "AndroidHiltConventionPlugin"
        }
    }
}
```

---

## Usage in Modules

```kotlin
// app/build.gradle.kts
plugins {
    id("myapp.android.application")
    id("myapp.android.compose")
    id("myapp.android.hilt")
}

android {
    namespace = "com.example.app"
    defaultConfig { applicationId = "com.example.app" }
}

dependencies {
    implementation(projects.feature.home)
    implementation(projects.core.network)
}

// feature/home/build.gradle.kts
plugins {
    id("myapp.android.library")
    id("myapp.android.compose")
    id("myapp.android.hilt")
}

android { namespace = "com.example.feature.home" }
```

---

## Anti-Patterns

- One giant convention plugin for everything — hard to apply selectively
- Duplicating Android config across module build files — defeats the purpose
- Not using Version Catalog inside convention plugins — version drift
- Business logic in convention plugins — they configure, not implement
- Not including `build-logic` in `settings.gradle.kts` — plugins not found

---

## Related Skills
- `gradle` — Gradle fundamentals
- `version-catalog` — used inside convention plugins
- `build-variant` — build type config in convention plugins
- `modularization` — convention plugins enable consistent multi-module setup
