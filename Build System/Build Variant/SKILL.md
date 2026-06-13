---
name: build-variant
description: >
  Android build types and product flavors for managing multiple build variants.
  Load this skill when configuring debug/release builds, multiple environments
  (dev/staging/prod), or feature-based flavors.
---

# Build Variant

## Overview
Build variants are the combination of build types and product flavors. Build types define how the app is built (debug vs release), product flavors define what is built (dev vs prod, free vs paid). Every combination produces a distinct variant — e.g., `devDebug`, `prodRelease`.

---

## Core Principles

- **Build types** = how to build (debug, release, staging)
- **Product flavors** = what to build (dev, prod, free, paid)
- Use **BuildConfig fields** for environment-specific values — not hardcoded strings
- Each variant can have its own `google-services.json` and resources
- Minimize the number of variants — complexity grows exponentially

---

## Build Types

```kotlin
android {
    buildTypes {

        debug {
            isDebuggable = true
            isMinifyEnabled = false
            applicationIdSuffix = ".debug"
            versionNameSuffix = "-debug"
            buildConfigField("String", "API_URL", "\"https://dev.api.example.com/\"")
            buildConfigField("Boolean", "ENABLE_LOGGING", "true")
        }

        release {
            isDebuggable = false
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
            buildConfigField("String", "API_URL", "\"https://api.example.com/\"")
            buildConfigField("Boolean", "ENABLE_LOGGING", "false")
            signingConfig = signingConfigs.getByName("release")
        }

        // ✅ Custom build type — staging
        create("staging") {
            initWith(getByName("release"))  // inherit from release
            isDebuggable = true
            applicationIdSuffix = ".staging"
            versionNameSuffix = "-staging"
            buildConfigField("String", "API_URL", "\"https://staging.api.example.com/\"")
            buildConfigField("Boolean", "ENABLE_LOGGING", "true")
            matchingFallbacks += listOf("release")  // fallback for dependencies
        }
    }
}
```

---

## Product Flavors

```kotlin
android {
    flavorDimensions += listOf("environment", "tier")

    productFlavors {

        // Environment dimension
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            buildConfigField("String", "ENVIRONMENT", "\"dev\"")
        }

        create("prod") {
            dimension = "environment"
            buildConfigField("String", "ENVIRONMENT", "\"prod\"")
        }

        // Tier dimension
        create("free") {
            dimension = "tier"
            buildConfigField("Boolean", "IS_PREMIUM", "false")
        }

        create("premium") {
            dimension = "tier"
            buildConfigField("Boolean", "IS_PREMIUM", "true")
        }
    }
}

// Resulting variants:
// devFreeDebug, devFreeRelease
// devPremiumDebug, devPremiumRelease
// prodFreeDebug, prodFreeRelease
// prodPremiumDebug, prodPremiumRelease
```

---

## Flavor-Specific Source Sets

```
app/src/
├── main/                  ← shared for all variants
├── debug/                 ← debug build type only
│   └── google-services.json
├── release/               ← release build type only
│   └── google-services.json
├── dev/                   ← dev flavor only
│   ├── java/
│   │   └── com/example/app/
│   │       └── DevTools.kt
│   └── res/
│       └── values/
│           └── strings.xml   ← override app_name for dev
├── prod/                  ← prod flavor only
│   └── res/
│       └── values/
│           └── strings.xml
└── devDebug/              ← specific variant only
    └── java/
```

---

## Variant-Specific Dependencies

```kotlin
dependencies {
    // ✅ Debug-only dependencies
    debugImplementation(libs.leakcanary)
    debugImplementation(libs.compose.ui.tooling)

    // ✅ Flavor-specific
    "devImplementation"(libs.debug.drawer)

    // ✅ Test
    testImplementation(libs.junit)
    androidTestImplementation(libs.espresso.core)

    // ✅ Staging matches release dependencies
    // (handled by matchingFallbacks)
}
```

---

## BuildConfig Usage

```kotlin
// ✅ Access in code
class ApiConfig {
    val baseUrl: String = BuildConfig.API_URL
    val isLoggingEnabled: Boolean = BuildConfig.ENABLE_LOGGING
    val environment: String = BuildConfig.ENVIRONMENT
    val isPremium: Boolean = BuildConfig.IS_PREMIUM
}

// ✅ Conditional behavior
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
}

if (BuildConfig.ENABLE_LOGGING) {
    OkHttpClient.Builder().addInterceptor(HttpLoggingInterceptor())
}
```

---

## Variant Filtering

```kotlin
// ✅ Exclude unused variants to speed up build
android {
    variantFilter {
        // Ignore staging + free combination — doesn't make sense
        if (flavors.any { it.name == "free" } && buildType.name == "staging") {
            ignore = true
        }
    }
}
```

---

## Anti-Patterns

- Hardcoding API URLs — use `buildConfigField` per build type/flavor
- Too many dimensions — variants multiply, CI time explodes
- Not using `matchingFallbacks` for custom build types — dependency resolution fails
- Different logic in code for dev/prod via `if (BuildConfig.FLAVOR == "dev")` — use polymorphism or source sets
- Committing production `google-services.json` to version control unprotected

---

## Related Skills
- `gradle` — build system fundamentals
- `version-catalog` — dependency management per variant
- `convention-plugin` — shared build type configuration
- `build-flavor-strategy` — deciding on flavor structure
- `ci-cd` — building specific variants in CI
